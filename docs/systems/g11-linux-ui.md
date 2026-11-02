# G11 Linux UI, rendering and development services

[Systems](README.md) · [G11](../devices/skydroid-g11.md)

## From hosted mixer to autonomous controller

The first October 23 proof ran actual EdgeTX calculations from pinned revision
`64c7221001332dba962c90abdf76ca9c89f8840b`, using an X7/QX7 native test fixture.
It passed 17 generated vectors and replayed 99 captured stationary frames;
40 partial-boundary bytes were discarded. The ARM Linux version matched host
results byte for byte on the G11. Generated timestamps and normalization were
bench assumptions, not measured acquisition latency. This headless demo used
no live controls or RF and temporarily suspended the stock app.

The later color UI used TX16S native configuration for bring-up, then separate
stock-input and custom-MCU backends. Neither simulator board name identifies
G11 hardware. The current research architecture moves acquisition, calibration
and mixing into the MCU; Linux provides UI, editable settings/model library,
networking, video and presentation. Its local mixer is disabled. Historical
Linux-mixer scheduling and freshness requirements must not be copied into the
autonomous architecture without preserving that change in ownership.

## Native rendering and settings

The UI renders at 1920×1080 with 3× layout scaling and fonts/icons regenerated
from upstream vector assets; an older 480×272 layout was retained for comparison.
LVGL uses Linux monotonic time, correcting an early clock that ran about ten
times slow. Eleven early redraw observations averaged 27.9 ms framebuffer
conversion with roughly 20 MiB UI RSS. Neither is an input-to-display bound.

Rendering respects actual fbdev geometry/stride. The internal buffer uses
RGB565 plus an 8-bit alpha plane, adding about 4 MiB total buffer memory over
the earlier arrangement; fbdev receives ARGB pixels. Touch follows the vendor
raw coordinates even though kernel limits are portrait and the framebuffer is
landscape. Injected native-coordinate navigation and real framebuffer reads
establish software behavior separately from physical finger-touch calibration.

Radio/model YAML lives separately from stock vendor configuration. Native YAML
descriptors must match the build's in-memory layout: the old six-slot serializer
was incompatible with the native eight-slot top bar. The deployed fix regenerated
matching descriptors and restored affected prototype fields from backup. A
successful UI build alone would not detect that persistence mismatch.

## Partial-redraw icon defect

October 25 horizontal noise in the logo and Model Setup icon was already present
in the packed RGB565A8 buffer, before framebuffer conversion or scan-out.
The pinned LVGL `fill_argb()` opaque masked-fill branch omitted a row-stride
adjustment: after a narrow dirty rectangle it advanced the mask only by the
rectangle width, reading the wrong mask position on the next row.

Adding `mask_stride - w` after each row matched the existing translucent branch.
For the captured logo rectangle `(1049,40)` through `(1142,74)`, the broken
address calculation predicted 3,289 of 3,290 thresholded pixels; correct stride
matched only 1,635. That connects the actual artifact to the code defect.
Tests compiled the real original and patched renderer under sanitizers:
original failed 32/48 split-versus-whole draw comparisons; patched passed all 48.
This is an offline causal test; the defect report alone did not qualify physical
post-deployment rendering. Later deployment captures provide separate evidence.

## Video controls and alpha

The Video page uses a separate worker for bounded Unix-socket transactions.
It validates replies and independently queries resulting state; failed source
commands are not automatically replayed. It can select stored primary/secondary
RTSP, UDP or stop, retaining existing URLs and the inspected fixed UDP 554.
It does not silently migrate stock configuration or create a boot preference.

Per-home-screen Video background is saved with the model and defaults off for
existing screens. It reveals the existing hardware video layer through the
background while widgets retain their own colors/opacity. Menus and transitions
restore opaque backgrounds. This toggle does not start a stream. A full-controls
layout is separate from the earlier small corner-panel video layout.

Real framebuffer and alpha captures verified UI settings and transparency data;
they do not contain the underlying video plane. Later
[decoder captures and coexistence tests](../experiments/g11-video-and-debugger.md)
established actual decoded pixels, while physical LCD composition remained open.
Mode acknowledgements and player `stream_active` flags must retain those limits.

## Wi-Fi management and persistent recovery

The UI talks to the existing per-interface supplicant socket rather than
starting another network manager. It offers scanning, hidden/open/WPA/WPA2
Personal networks, masked password entry, connect and forget. Explicit actions
change saved profiles; failed joins attempt to restore the previous connection.
A changed network requests release/renewal from the verified existing DHCP
client. Enterprise, WEP, WPA3-only, hotspot and static-address modes were not
offered by this implementation. These are its feature boundaries, not universal
chip capabilities.

A native nl80211 helper was needed because the driver's legacy `iwconfig`
power-save setter was unsupported. Initial query already reported power saving
off; setting it again did not remove intermittent approximately 200 ms ping
delays. The flag did not establish the vendor firmware's internal sleep state.

An early temporary guard checked only power saving. The October 24 persistent
boot service superseded it: after vendor startup, it enabled wlan0, reused
healthy supplicant/DHCP processes, started missing ones, reconnected enabled
saved networks and checked power saving every five seconds. It preserved
credentials, priorities and disabled-network choices. A lock prevented duplicate
workers; target BusyBox timeout syntax and zombie-process behavior were handled.

Starting twice retained one worker. A detached hardware test terminated the
interface's supplicant and DHCP client; recovery was observed on the fifth
two-second sampling interval plus command time, with saved association, IPv4
and power saving off, without Linux reboot. A subsequent MCU system reset
rebooted Linux and verified automatic service startup. This supersedes earlier
README statements that Wi-Fi support ended at reboot. Driver-load failure,
unavailable APs and invalid credentials remain outside the service's recovery.

## Linux executable delta updates

The UI update tool derives an additive binary patch from exact old/new files,
then packages bounded copy/add operations in gzip so the target needs only its
existing decompressor and a small helper. The helper writes an exclusively
created separate output, checks format/size/operation bounds and truncated or
trailing data, flushes output and removes partial output on handled errors.
Abrupt termination may leave a partial file.

The coordinator, not the helper, verifies SHA-256 of old executable, helper,
compressed patch and reconstructed result. Staging on the destination filesystem,
preserving the old binary/settings and cleanly stopping/restarting the known
owner make the result reviewable and recoverable. Successful reconstruction
alone does not prove the installed UI works: processes, UART ownership, settings
and actual framebuffer were checked separately. Failed transfer/reconstruction
must leave the running executable intact. This is a Linux-file workflow, not
an MCU bootloader or flash-recovery mechanism.
