# Direct SurfaceView output: experimental device trial

The `direct_surface` preference defaults to `false`. When enabled, mirroring
decodes directly into the display SurfaceView instead of blitting through the
view-sized GL window. This can preserve video dimensions on TVs with a 4K panel
and 1080p Android UI, provided the hardware composer accepts the video layer.
It does not force the sender to transmit higher resolution.

Set via the existing DUMP-protected SettingsReceiver, then restart the server:

```sh
adb shell am broadcast -n io.github.jqssun.airplay/.SettingsReceiver \
  -a io.github.jqssun.airplay.SET_SETTING --es key direct_surface --es value true
```

For a separate applicationId, use the actual package before `/` and the full
`io.github.jqssun.airplay.SettingsReceiver` class name after it. This repository's
normal package and build settings are unchanged by the trial feature.

## Validation

1. Connect a sender and leave its Home screen visible, without opening Photos
   or rotating first. Verify hardware composition on this initial connection
   as well as after subsequent display-mode changes.
2. Generate sender activity, then pause on small text or a detail pattern.
3. Record decoder dimensions and capture `adb shell dumpsys SurfaceFlinger`.
4. Run `python3 tools/check_render_path.py dump.txt --decoded-size WIDTHxHEIGHT`.
   For a separate package, also pass `--package PACKAGE`.
5. Compare visual detail, sound and frame pacing. Exercise fullscreen changes,
   rotation, background/resume, disconnect and reconnect. Verify default GL mode
   still behaves as before.

The checker requires one visible matching buffer and its uniquely associated
HWC entry, full source dimensions, and Device/Sideband composition. A pass proves
those properties in the dump; physical panel detail still requires visual or
hardware-specific validation. A full-sized Client buffer is insufficient when
the client target is only 1080p.

## Initial connection on MStar

Configuring the decoder against the GL SurfaceTexture, then moving it with
`setOutputSurface()`, retained buffers that required Client composition on the
MiTV-MSSP1. Despite a full-sized 994x2160 crop, those buffers were composited into
the 1080p client target. A new decoder configured directly against SurfaceView
used Device composition. This explains why a Photos/rotation round trip could
improve the Home screen.

Direct mode now retains the first keyframe while waiting up to five seconds for
SurfaceView creation. The native frame thread releases the renderer monitor
while waiting, so the UI can attach its surface. Stop, resolution changes and
server shutdown cancel obsolete starts; shutdown disables further frame starts
before joining native threads. If no view appears in time (for example while
remaining in the background), the existing GL path is used and a warning is
logged. The timeout path does not guarantee hardware composition.

## Known limits before an upstream release

- Some vendor decoders cannot switch between SurfaceTexture and SurfaceView
  usage. Initial direct configuration retries the original GL hardware path;
  failed switching from an existing direct surface may require reconnecting the
  sender for a fresh keyframe.
- Preview/fullscreen/tab/PiP UI branches now use a single attached SurfaceView.
  Its viewport follows the video slot; changing a tab retains the previous
  viewport under the opaque UI instead of destroying the native surface.
- Android can still destroy the native surface when the whole activity/window
  becomes hidden or is recreated. A fully static sender may then require a new
  frame on return. Retaining a UI view does not promise retention across OS
  window destruction; background/resume and activity recreation remain explicit
  validation cases before claiming equivalence with the GL texture path.
- Intermittent green/block-corrupted HEVC frames were observed on both the earlier
  deployed direct-output build and the current upstream-based trial. A process restart/reconnect restored normal
  video; full buffer dimensions and Device composition alone did not detect the
  corruption. Its root cause is not established, so this trial does not claim
  to fix that separate rendering/decoding failure.
- The mode is experimental, opt-in, and has not been validated across devices.

Baseline on Xiaomi MiTV-MSSP1 / Android 9 / PI.5925: HEVC stream 994x2160
(portrait) reached a 497x1080 RGBx GL buffer with Client composition. Panel
properties report 3840x2160 while Android's display and client target report
1920x1080. Landscape stream was 3840x2160. These are baseline observations, not
results of the direct-output trial.

## Related implementation

[PR #60](https://github.com/jqssun/android-airplay-server/pull/60), revision
`d6d75e7`, already proposes direct output and a retained full-window view for
HiMedia H10 / Android 7. The retained-view approach here is informed by that
work. This trial targets current upstream v0.0.36 and exposes an opt-in runtime
preference instead of H10-only build types. Its retained view follows the
actual aspect-correct preview/fullscreen slot, while the other UI stays opaque.
The startup wait and EGL-detach handshake address the measured Xiaomi first-
connection path; the comparison is not a claim that it outperforms the H10
implementation on other hardware or fixes its audio synchronization issues.

## Native view continuity regression (Android TV)

Build `assembleDebug assembleDebugAndroidTest` and install the debug receiver
and its test APK on an idle test TV. Stop competing receivers. Enable
`direct_surface` and `idle_preview` with the existing SettingsReceiver. Wake
and unlock the display first: an active screensaver owns focus and can hide
SurfaceView even though the activity's decor reports that it is shown.

```sh
adb shell input keyevent KEYCODE_WAKEUP
adb shell am instrument -w io.github.jqssun.airplay.test/io.github.jqssun.airplay.ui.SurfaceContinuityInstrumentation
```

Use the actual test package for separately packaged builds. The test starts
Overview, binds the real service, and uses the TV remote navigation from the
initial Stop focus: Down to Settings, Left to Logs, then Left back to Overview.
It verifies each selected tab through the real Compose semantics tree, then
checks the renderer's actual Surface object, validity and SurfaceHolder lifecycle
callbacks, without adding production test hooks or mocking a codec. It requires no sender. With direct
mode enabled, the old UI fails because Logs destroys the surface; the retained
view must remain the same valid object, without any SurfaceHolder destruction
or creation after the initial attachment. This catches native replacement even
when Android reuses its Java Surface wrapper. Passing `-e skip_navigation true`
intentionally leaves Overview selected and must fail the Logs assertion.
With direct mode disabled, the test only
checks that the normal preview is restored. This test does not prove retained
pixels, hardware composition, codec switching or physical-panel resolution;
those need a real stream and the separate dump/visual checks above.
