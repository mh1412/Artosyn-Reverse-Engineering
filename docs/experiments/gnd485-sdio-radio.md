# Gnd485 native SDIO radio bring-up

[Experiments](README.md) · [AR9311 platform](../soc/ar9311-platform.md) · [Radio loading](../firmware/ar8030-loading-and-driver.md)

The October 27–30, 2026 work established source-built Linux/Open8030 association,
application command acknowledgements and video reception on the standard Ascent
VRX. It exposed board power/reset requirements and several misleading success
indicators. Receiving bytes, clean selected-picture decoding and a complete live
ground-station pipeline remain distinct milestones.

## Board controls and enumeration

| Resource | Gnd485 finding |
| --- | --- |
| SDIO host | `0x01b00000`, GIC SPI 48 |
| Radio reset | GPIO3, active low; the reference goggle's GPIO23 does not apply |
| Radio I/O enable | GPIO12 |
| Radio I/O voltage | GPIO35 selects 3.3 V |
| RF enable | GPIO41 must be high |
| Separate SD-card power | GPIO14 must have an explicit consumer |
| USB mode pins | GPIO9/10 explicitly owned as board hogs |
| SDIO clock | Matched stock selection 87, phase/configuration 2 |
| Running radio identity | SDIO `4152:8031` |

The initial GPIO driver initialized direction shadows as all-input. Accessing
bank 0 for radio setup dropped inherited SD power, blocking mounting. Explicit
SD/USB consumers fixed that dependency; later readable-direction preservation
improved handover as described in the [platform study](../soc/ar9311-platform.md).
The first blocked-mount trials were followed by a clean read-only filesystem check.

Command tracing showed ROM CMD5/3/7 and CIS reads succeeding after RF enable.
The ROM's incomplete CIS lacked `max_dtr`; without the high-speed capability,
Linux then turned its clock off. Adding the capability resolved that specific
enumeration boundary. Temporary pinmux probes failed and were restored; they
are not part of the working recipe.

Generic MMC power sequencing reset the radio during expected post-upload
re-enumeration, discarding RAM firmware and returning to ROM. A one-time reset
provider keeps reset released across host power-off. The final cold boot uploaded
once and reached the running identity. Later clock-provider tests separately
established ownership of source selection, phase and gates; successful firmware
upload alone was not proof that the clock handover was complete.

## Firmware and application boundaries

The initial tested combination used ar_libre commit
`5c66a0f7363e5a576a8f9fce3667805b7e39d426` and Open8030 commit
`89d3c634c11d5665dcc17401c5d8b3172bbacf8b`. Independent and Buildroot compilation
matched the 133,632-byte radio image:

```text
radio image  b8d4d456000d01c7a04ad32b809e5a6ea4ae88fdb27405dc3ab96c072e3446b1
Gnd485 profile 1bc673f086b67f2a742233619d655d92aca6816932a57e2b674cbdbd4f6f12c5
shared config 11a3c82b58106f61f4ac7a65bfb6916c4c96260f0d8f63db4aa39c3c86f8ef4c
```

The shared configuration is the published VRX Pro ground JSON. Native Gnd485
SDIO still requires its own board profile; sharing JSON does not make the USB
profile or forced-USB transport appropriate.

The loader preinitializes the ground role, so client INIT returns `-7` in that
state. The bench client accepts it only with subsequent successful START and
valid DEV status. It closes sockets and explicitly STOPs/DEINITs; reopening
passed. A bare INIT error is neither sufficient failure diagnosis nor permission
to skip later state validation.

The bounded C receiver adapts Kestrel startup commands, requesting 1080p60 and
the displayed 25 mW setting. Ten CRC-valid acknowledgements establish those
application transactions; they do not measure emitted power or complete camera
configuration effects. Binding, persistent peer selection, live picture assembly,
IDR recovery, UI, OSD and recording belong to the separate ground application.

## Reception and loss accounting

The final initial cold-start run associated `0 → 1 → 2`, received 34,886,004 raw
video bytes in 25 seconds and acknowledged all ten commands. Sampled socket
drops and bus/RX/TX errors were zero; firmware uploaded once. Source headers
reported 1920×1080, 60 fps and four slices per picture.

Nevertheless, the capture contained **two incomplete framed packets**, plus
the expected incomplete tail at stopping, and offline HEVC decoding reported
POC/reference errors. Accepted packet XORs passed. The remaining gap location
and cause were unresolved; zero sampled counters did not establish loss-free video.

An earlier blocking startup probe overflowed an attached kernel socket queue
by 505 frames. The global `rx_dropped` statistic did **not** count those socket
drops. The packaged client instead drains data while scheduling one startup
command per 250 ms. That corrected a known host scheduling problem without
explaining every later framing gap.

| Later test | Recorded result | Limit |
| --- | --- | --- |
| October 29 canonical board regression | Ten ACKs, about 26.2 MB received; six bad tails | Not loss-free video |
| October 30 shared-radio configuration | 14,925,196 bytes in 20.002 s, ten ACKs, no rejected/invalid responses | Framed transport observation; no whole-stream decoded-video assertion |
| October 30 combined regression | 13,865,244 bytes in 20 s, ten ACKs, zero reported bus errors/bad framing | Bounded clean window, not a general gap fix |

The shared-radio change made transport, role, profile and factory policy
board-configurable, with canonical image/config loader names. Its Ascent firmware
and configuration hashes stayed identical. VRX Pro/Lite and a synthetic other
AR9311 board passed configuration/recipe checks in that test; those were not
additional hardware runs.

## Decoder and colorimetry checks

A deliberately selected intact eight-picture VPS/IRAP-starting window decoded
on Wave521C to SHA-256:

```text
d6f98b2c2d4f957380ba876fb2990a56901c15034a43570ad43d69c2c6aa4479
```

This exactly matched FFmpeg when preserving full-range `yuvj420p`. The first
limited-range `yuv420p` reference had converted levels and was invalid for byte
equality. The source metadata was full-range BT.470bg, whereas the examined
player used limited-range BT.709 conversion. Correct color metadata propagation
remained application/display work.

The selected window explicitly excludes damaged periods and does not validate
the entire capture. Offline DMA playback used zero CPU pixel copies, but the
connector reported disconnected; completed forced scanout is not independent
visual proof. Its eight-frame average includes startup and is not sustained
throughput evidence.

## Remaining validation

The platform provides radio loading, SDIO and bounded receiver/media exercisers.
Complete-picture validation, gap localization through firmware/socket transport,
reconnect recovery and live decode/display orchestration still need their own
tests. Command ACK integrity, packet XOR, stream syntax, pixel equality and
visible output each validate a different boundary. None alone establishes a
complete ground-station implementation or glass-to-glass latency.
