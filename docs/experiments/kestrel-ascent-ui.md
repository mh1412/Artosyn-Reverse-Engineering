# Kestrel UI and HUD validation on Gnd485

[Experiments](README.md) · [UI architecture](../systems/kestrel-software-ui.md)

The October 8 bring-up and subsequent renderer checkpoints tested a software
overlay on stock Gnd485 18.21.10, build 202608141800. Radio/media remained in
the original backend processes. These are recorded experiments, not a new
hardware run performed while preparing this wiki.

## Initial no-link framebuffer test

The static ARM64 LVGL executable was approximately 2.13 MiB stripped, with no
dynamic section or GLES/EGL/GBM/DRM dependencies. It temporarily replaced
GlassesUI in RAM. The initial periodic-status parser rejected 297-byte payloads
until the [18.21.10 layout](../protocols/ascent-stock-msp.md) was established.

The final 20-second no-link run recorded:

| Measurement | Result |
| --- | --- |
| Visible framebuffer | 1920×1080, 32 bpp, stride 7680, offset 0/0 |
| Channel offsets | R16 / G8 / B0 / A24 |
| Mapped allocation | 24,883,200 bytes, existing three-height stock allocation |
| Status updates | 49; zero unknown/rejected frames |
| RF state | Disconnected; raw MCS 0 treated as invalid |
| Receiver supply / CPU | 11,352 mV / approximately 48.36–48.89°C |
| Peak RSS | 19,352 KiB |
| UI ticks / partial flushes | 480 at a 25-Hz configured limit / 68 |
| Final pixel readback | 2,073,600 pixels, zero mismatches |
| Backend/radio processes | Original PIDs retained |
| Stock GUI restoration | Successful in this run |

An intermediate `EADDRINUSE` failure occurred before framebuffer access and
restored GlassesUI. A bounded endpoint-acquisition retry was added. The sub-1-ms
all-tick average included idle time and was not an animation or video benchmark.
The transparent overlay preview was not a capture of the underlying video plane.

## Establishing actual live video

Powering the VTX initially produced connected status and MCS around 6–10, but
decoded-frame progress stayed zero and passive capture showed no compressed
video flow. Restoring the stock UI, rebooting the receiver and cycling the VTX
did not restore video. Camera initialization remained pending and the encoder
running flag stayed clear.

The user then updated the VTX and confirmed stock video. The prior VTX version
was not established, so the exact incompatibility was not identified. The
receiver remained on 18.21.10 with backend SHA256
`243c11864d266caebdc76d9c4357ca6517fd1afdbbfcbaf9d6c59c113d6d6216`.

Offline inspection established `g_vdec_stats` at `0x6ac060`, decoded counter
at `+0x68` and float FPS estimate at `+0x6c`. Sampling was read-only, without
debugger attachment or process pause:

| Phase | Counter | Approximate three-second increment |
| --- | --- | ---: |
| Stock UI | `0x341d → 0x34d1` | 180 |
| Kestrel UI | `0x4ac7 → 0x4b7c` | 181 |

The FPS estimate stayed near 60 and the air-unit flags became `0x03`, including
encoder-running. The user explicitly confirmed video and HUD on HDMI. That
observation establishes practical coexistence in this session beyond process
liveness and framebuffer readback.

## Perspective, menus and animation

CPU-projected Chakra Petch text was then tested over live video. A 20-second
run recorded 49 status updates, no invalid frames, 19,696-KiB peak RSS, roughly
200 KiB of projected masks / 35 rasterizations, and zero pixel mismatches.
The user confirmed improved readability. The approximately 1.9-ms all-tick
average again included idle time.

Device menu tests exercised theme preview/cancel/apply, cadence rollback and
both menu sides, with preference readback. The decoder advanced 180 frames
over approximately three seconds. Mapped stock key packets and user feedback
validated the menu subset, not every physical record/bind input.

Animation tests used two 800-MHz Cortex-A53 cores, the performance governor and
a live RF link. Six-second offscreen tests compared matching Release builds:

| Variant | Average | p95 | Maximum | One-core CPU | Skipped deadlines |
| --- | ---: | ---: | ---: | ---: | ---: |
| Earlier direct drawing | 21.32 ms | 55.77 ms | 80.75 ms | 38.93% | 93 |
| Cached opaque menu / edge damage | 4.48 ms | 24.40 ms | 68.41 ms | 13.99% | 41 |

These exclude framebuffer/HDMI presentation. The later on-screen sequence of
20 open/fold cycles averaged 6.411 ms, p95 32.897 ms and maximum 122.191 ms,
with 1,115 decoded frames over approximately 19 seconds. Pixel readback remained
exact. A subsequent fix shrank the closing reveal to zero height, eliminating
the known geometry discontinuity; its measured p95 was 38.09 ms. Average cost
improvement did not establish a guaranteed 50/60-Hz UI or absence of tearing.

## Passive worker recovery

The later split UI/HUD checkpoint was observed over working video and valid
STATUS/ANALOG telemetry. Descriptor inspection found framebuffer, backend and
input endpoints only in the UI; the HUD had its private socket/shared memory
and diagnostic output.

Killing the HUD during a theme preview preserved the UI process. Cancel restored
the previous theme, navigation continued and the restarted worker received
current data. Backend/radio PIDs were unchanged; the decoder advanced 208 frames
during approximately 3.5 seconds. The user confirmed the displayed video/UI.

An eight-second closed-menu Canopy sample measured UI/HUD CPU at 6.48%/1.62%
of one core, or 4.05% combined across the two-core chip. Combined PSS was
33,496 KiB (about 32.7 MiB). These values exclude the video/backend processes.
Shared-page RSS must not be added as though all memory were private.

Vector's seven-region renderer initially used 8.11% UI plus 6.99% HUD one-core
CPU. After static-region reuse, an eight-second sample measured 7.72% plus
4.61%, or 6.16% combined chip CPU. One worker update took 7,759 µs versus
roughly 42–53 ms before reuse. These small workload-dependent samples do not
guarantee frame cadence. Independent worker replacement preserved UI/menu state;
overlay readback and user observation verified the displayed layout.

## Restore incident and limits

After the serial-decoder test exited with exact pixel readback, the subsequent
stock restore/relaunch sequence left a blank/frozen display and SSH connections
without banners. The cause remained unresolved. A directed three-second power
cycle restored stock boot; RAM binaries/assets and last observed preferences
were restaged. Later in-session replacements paused the original restoration
wrapper to avoid its framebuffer-module reload during handover.

Normal wrapper cleanup is therefore a tested path with a recorded failure,
not a production recovery guarantee. Full UI feature parity, VTX reconnect,
resolution changes, latency/tearing and all controls remain unqualified.
The later [GE2D benchmark](../soc/ar9311-ge2d.md) had a stationary decoder counter;
its retained FPS number must not be used to claim active-video acceleration
performance.
