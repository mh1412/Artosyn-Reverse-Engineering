# System architecture and video paths

[Systems](README.md) · [Documentation index](../../README.md)

## Common roles

An Artosyn-based video system usually contains several independently programmable
components. The application host handles camera/media or display/UI work, while
a companion radio moves data over the wireless link. Controllers can add an MCU
for physical controls and board management.

```text
Camera → host capture/encoder → radio → RF → radio → host decoder → display
                  ↑                          ↑
           configuration             link control / telemetry
```

This diagram describes roles, not a universal packet format. The host CPU, radio
transport, media framing and firmware differ by product.

## Ascent air and ground

The standard Gnd485 receiver uses an AR9311/AArch64 Linux host. The matched
17.5.8 H Sky/Lite transmitter firmware uses an ARMv7/musl host with a Hi3516CV610
SDK profile. Both use separate SDIO radio firmware and a low-delay video backend,
but their existing host binaries and board support packages are incompatible.

The firmware comparison found matching names for 21 selected public `bb_*`
radio APIs and five additional helpers. These show interface lineage, not
source-code identity or binary ABI equivalence.

On the receiver, compressed video can be recovered from the existing
`daemon_sdio`/backend TCP path before decoding. Passive extraction demonstrated
host HEVC playback and provides a foundation for a USB/network ground station.
It does not require capturing the framebuffer or re-encoding decoded pixels.

See [Gnd485](../devices/ascent-vrx.md) and [Sky482](../devices/ascent-lite-vtx.md).
The [detailed air/ground comparison](ascent-air-ground-comparison.md) covers
media/ABI boundaries, radio-image differences and the requirements for a
headless receiver on transmitter hardware. The
[ground-station record](ascent-groundstation.md) covers the later persistent
VRX proxy and direct host playback.

## BetaFPV-related air/ground pipeline

P1 SKY/GND and VR04-related research describes a different media pipeline with
an IP tunnel over the radio's SDIO interface. Product labels should retain their
board/firmware context rather than being assumed to be exact aliases.

See the [reference-board study](../devices/betafpv-vr04-reference.md) for physical
interfaces and [SDIO/IP transport](../protocols/vr04-sdio-transport.md) for the
radio's host-facing loader, netdev and receive-resynchronization behavior.

The recorded VR04 1080p mode carries two independent HEVC bitstreams: a top
1920×560 tile and a bottom 1920×552 tile. A 32-row overlap allows the two decoded
images to be combined into the displayed picture. Other modes use different
tile patterns.

Media setup and receiver readiness matter. The observed stream sends its initial
IDR at session start; a consumer that misses that point cannot assume a periodic
IDR will soon repair its reference state. Packet arrival and link association
alone therefore do not prove a usable video session.

The [media-session study](../protocols/vr04-video-session.md) documents the
104-byte parameter reply, video header, mandatory user-data SEI and timestamp
rebasing. [Link-control policy](vr04-link-control.md) separates radio association,
consumer readiness, rate adaptation and standby.

Air-side camera research identifies an NT99235 sensor and an Artosyn
CSI/VIF/ISP/CVISP pipeline. That camera path is specific to the studied air-unit
platform and must not be transferred to the Hi3516CV610-profile Ascent Lite host.

The later [standard P1 Air investigation](../devices/betafpv-p1-air.md) identifies
an SC231AI sensor. Its source stack sends complete-frame 1080p HEVC through
Kestrel and native arlink; it does not simply reproduce the reference tiled
vendor pipeline. The [camera study](../soc/ar9311-p1-camera.md) documents its
sensor/ISP differences and buffer ownership, and the
[integration experiments](../experiments/p1-source-stack.md) establish matched
radio delivery and host/receiver decode agreement.

## G11 controller integration

The G11 combines a SigmaStar Linux UI/video host with an MCU control path and
an internally USB-connected AR8030. In the custom architecture, the MCU owns
physical acquisition and mixing while Linux supplies model configuration, UI,
networking and video.

This division allows control processing to be tested independently of Linux
reboots. Physical RF/control delivery still requires separate validation beyond
the mixer loop and transport counters. See the [G11 study](../devices/skydroid-g11.md).

## Measuring the path

Keep each measurement boundary explicit:

| Measurement | Establishes |
| --- | --- |
| Bytes delivered by radio | Transport activity over the measured interval |
| Successful host decode | Usable compressed data for that captured stream |
| Decoder output hashes | Agreement with the reference for the tested pictures |
| DRM flip events | Display-buffer presentation timing |
| Receiver input-to-flip timestamps | Receiver-side pipeline latency |
| Optical camera-to-display test | End-to-end glass-to-glass latency |

A 60 Hz display, 60 fps metadata or a 30 fps capture adapter cannot individually
establish the end-to-end latency or actual radio frame cadence.
