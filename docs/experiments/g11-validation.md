# G11 development stages and qualification boundaries

[Experiments](README.md) · [G11](../devices/skydroid-g11.md)

The G11 research progressed from Linux-hosted EdgeTX to autonomous MCU control.
Its implementation plan retains several dated “current” audits; their statements
apply to different stages. This page connects those stages to the detailed
evidence and identifies what remains unqualified.

## Architecture and recovery history

| Stage | Recorded progress | Boundary |
| --- | --- | --- |
| October 23 | Headless/native EdgeTX fixtures and initial UI work | No physical replacement MCU or end-to-end receiver result |
| October 24, early | Real native framebuffer/menu deployment, stock controls and video IPC | A player response or UI image did not establish decoded video |
| October 24, resident firmware | Matching flash/option backups, custom startup, UART uploads, watchdog and partial/non-starting application recovery | Linux supplied mixing; external CRSF remained unqualified |
| October 25, direct boot | Serial-resident requirement superseded by SWD recovery; memory qualification and autonomous integration | Firmware generations require their own image/layout records |
| October 25–26, autonomous model | MCU acquisition/calibration/mixing, durable selected model and runtime; Linux observation/editing | RF-off autonomy does not establish connected receiver control |
| October 26, service deployments | Module bridge, sensors/transforms, trims/resets, clocks, presentation and decoded-picture checks | Each has distinct simulated and physical validation |
| October 26, connected electrical follow-up | RX/PPM output located, unloaded active drive improved, module-related faults/resets recorded | Connected replies and receiver delivery still not established |

The historical Linux-mixer session expiry near 100 ms and its maintenance/RUN
reconnect procedure are not the autonomous control contract. Later Linux loss
must leave locally acquired MCU control running. Likewise, an older requirement
to retain serial/vendor updating does not describe the later direct SWD profile.
See [UART/session history](../firmware/g11-uart-and-session-lifecycle.md) and
[MCU update history](../firmware/g11-mcu-update-history.md).

## Evidence by subsystem

| Subsystem | Recorded evidence | Remaining qualification |
| --- | --- | --- |
| Board/memory | Restoring 128 KiB flash and 16 KiB SRAM tests; segmented image and active mirror readback | Retention/endurance, upper-fetch timing and full environment range |
| Inputs/engine | Confirmed movement map; native/ARM comparisons; selected physical models near 500 Hz | Final physical calibration, every supported model's timing, full feature compatibility |
| Linux independence | UI pause/exit/replacement and Linux reboot on RF-off bench; model restored after MCU reset | Continuous physical receiver controls during those events |
| Model persistence | Exact-identity journal/restore tests, physical selected-model reset and cache/mirror operations | Physical interrupted writes and complete shutdown/power-cut recovery |
| UI | Native framebuffer/menu and source-marker checks; remote observer with zero Linux mixing | Remaining native bindings, live editing and all physical interaction combinations |
| Video | Two captured 720p decoder pictures exactly match source frames; finite coexistence with mixing | Physical LCD composition, original fallback restoration and broader stream/endurance tests |
| Module/telemetry | Actual Lua/UI/ARM configuration and transformed sensor tests with synthetic replies | Real Nano discovery/parameters, sensor traffic, receiver channels/arming/failsafe |
| Power/battery | Stock code reconstruction, voltage/charge observations, prepare/cancel and watchdog recovery | Full deliberate off/button wake, rail/charging transitions and absolute voltage accuracy |
| Feedback/audio | MCU pattern/intent tests, deployed sound assets and silent PCM drain | Audible speaker output, motor wiring/actuation and complete audio behavior |
| Clocks | Backward-time wait fix, physical edits and short MCU holdover | Oscillator accuracy and RTC retention after power removal |

## October 26 deployment and readback checks

Three retained installation records connect service changes to specific MCU/UI
pairs. These are historical checkpoints, not interchangeable firmware labels.
Hash prefixes below identify files whose full SHA-256 values are in the
[artifact manifest](../../assets/experiments/g11-service-deployments/manifest.json).

| Checkpoint | MCU ELF / final UI SHA-256 prefixes | Schema | MCU boot |
| --- | --- | --- | --- |
| Flight reset | `ad324649…` / `6af67046…` | `0xeb04e1ae06c8dfff` | 121 |
| Numeric telemetry | `36af66b3…` / `31038c89…` | `0xeddcc26d6cb570ec` | 122 |
| Presentation and corrected battery alarm | `b14c2827…` / `b38e9c84…` | `0xeddcc26d6cb570ec` | 123 |

An offline comparison of the retained binary artifacts confirmed, for each
checkpoint:

- The planned 128 KiB image equals the installer's expected and verified images.
- The later 128 KiB readback differs from that plan at exactly two bytes, inside
  the [boot nonce journal](../firmware/g11-mcu-memory-and-models.md#fast-and-stopped-mode-regions).
  The offsets are `0xf910–0xf911`, `0xf912–0xf913` and `0xf914–0xf915`, respectively,
  relative to flash base `0x08000000`.
- The retained option-byte copies match, and all 334 saved working-state bytes
  match before and after the update. These byte comparisons do not establish
  every model's behavior or interrupted-write recovery.
- The flight-reset post-boot image is exactly the numeric-telemetry preflight
  image; that update's post-boot image is exactly the presentation preflight
  image. This corroborates the recorded order independently of archive names.

First-boot reports explicitly left flash verification pending; the later
quiesced readback records close that check. Their full-file identities were
rechecked here. This is analysis of retained captures, not a new SWD readback
or a power-loss-atomic installation guarantee.

The physical flight-reset check used an idle model with zero timer, trim, GVar
and logical-switch state. Nonzero-state behavior remained a separate simulated
check. Numeric telemetry was deployed without a live Nano telemetry source.
Presentation initially used UI `8fae0786…`; its flash/holdover observations
precede the UI-only battery-alarm correction to `b38e9c84…`. The correction kept
MCU boot 123 but changed the UI process and management generation. A single
aggregate PASS must not assign every subtest to the final UI hash.

All three physical records kept RF unconfigured/off and Linux mixer cycles at
zero. Their clock, model restoration, clean framebuffer and returned control
observations do not establish connected-module replies or receiver delivery.
The [published home-screen capture](../systems/g11-linux-ui.md#native-rendering-and-settings)
and [audio correction](../firmware/g11-feedback-and-audio.md) preserve those
separate evidence boundaries.

## Corrections that change interpretation

- Five-minute UART console-mitigation passes did not establish loss-free transport:
  a later 90.22-second run added an overrun/reject. Live debugger reads also caused
  repeatable management disruption in separate experiments.
- The early static-high capture and assumed TX contact did not identify the final
  output route. Later scope work found output on RX/PPM, with connected-module
  reset failures remaining after unloaded edge improvement.
- An earlier video coexistence report said idle mode was restored. Later actual
  decoder captures showed mode 0/inactive status while resolution remained 720p;
  original fallback-picture restoration was not proven.
- Clock-only, telemetry-only and reset-capability paragraphs were superseded by
  later paired deployments. Schema compatibility and active model contents must
  accompany each capability claim.
- Revision-14 numeric telemetry left 380 bytes of fast flash in that recorded
  build; the subsequent boot-123 presentation image left only 32 bytes. These
  are particular image margins, not capacity figures for all later builds or a
  claim of complete runtime/stack qualification.
- The [model-audio checkpoint](../firmware/g11-feedback-and-audio.md#undeployed-model-audio-checkpoint)
  was neither fully tested nor deployed. Its prepared archive name and passing
  model-restoration comparison do not establish an installed audio feature.

The latest electrical failures are detailed in
[CRSF experiments](g11-crsf-electrical.md). [Video/debugger experiments](g11-video-and-debugger.md),
[model memory and commits](../firmware/g11-mcu-memory-and-models.md),
[calibration](../firmware/g11-calibration.md), [clocks](../systems/g11-clocks.md)
and [feedback/audio](../firmware/g11-feedback-and-audio.md) preserve their own
artifact identities, methods and limits.

The complete retained October 22–26 development log has been reconciled with
these articles, including failed attempts and later corrections. Documentation
coverage remains incomplete: that review does not independently validate every
supporting capture, source implementation or hardware claim.
