# Protocol layers and recovered formats

[Protocols](README.md) · [Documentation index](../../README.md)

## AR8030 USB host link

The studied USB host-link implementation distinguishes a boot-ROM device
(`4152:8030`) from running radio firmware (`1d6b:8030`). Firmware/config upload
uses vendor control requests; the running device exposes bulk OUT `0x01` and
bulk IN `0x81`. These observations describe the USB implementation, not the
Gnd485 SDIO enumeration.

The running protocol's recovered message prefix is:

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0 | 1 | Marker `0xAA` |
| 1 | 4 | Payload length, little endian |
| 5 | 4 | Request ID, big endian; domain in the high byte |
| 9 | 4 | Message ID, big endian; request/reply matching |
| 13 | 4 | Signed status, big endian, in replies |

This is a prefix overview, not a complete serializer specification. Message
payloads and complete framing/checksum rules need their own tables. USB transfer
boundaries must not be assumed to correspond one-to-one with messages.

The [host-link article](ar8030-host-link.md) supplies the complete logical frame,
XOR/trailer, socket completion rules, decoded requests, events and telemetry limits.

## Wireless control and MAC

Recovered AR8030 behavior includes acquisition, login, logical peer slots,
binding, MCS negotiation, timing and recovery. AP and DEV are radio roles;
they do not establish Wi-Fi interoperability.

The exposed MAC bytes and PHY register fields describe only the investigated
subset. Preambles, complete coding/interleaving, waveform generation and all RF
timing modes remain separate physical-layer questions.

## Ascent receiver media boundary

The Ascent receiver's existing loopback TCP connection carries vendor-framed
compressed video between `daemon_sdio` and the backend. TCP reassembly and vendor
deframing recovered HEVC suitable for host decoding. This is a host/backend
interface, not a specification of the on-air waveform.

See the [extraction result](../devices/ascent-vrx.md#compressed-video-extraction).

## BetaFPV-related UDP video

The studied P1/VR04-related pipeline uses UDP port 10000 for telemetry/media setup
and port 10001 for video, with a separate identity handshake on port 20001.
Video travels over an IP tunnel carried by the SDIO radio interface.

The recovered video datagram has a 36-byte little-endian header, followed by the
declared HEVC payload length and a four-byte tail marker. Header fields include:

- Start magic `0x12345678`, stream length and channel index.
- IDR indication, frame ID, millisecond timestamp and resolution.
- Tail magic `0x87654321` and CRC-32 covering the first 32 header bytes.

The recorded VR04 1080p mode interleaves two independent tiled bitstreams. Its
session-start IDR behavior makes receiver readiness part of successful startup.

Compatibility also depends on bitstream details: the inspected vendor receiver
byte-checks the beginning of an IDR for a VPS followed by an SPS at fixed offsets,
and the recorded per-frame prefix SEI is required by that pipeline. A valid HEVC
stream for a general-purpose decoder is therefore not necessarily accepted by
the vendor application. These constraints should retain their specific firmware
scope rather than being treated as universal Artosyn requirements.

## G11 custom MCU protocol

The custom Linux/MCU protocol is separate from the stock Skydroid UART format.
Its decoded packet contains little-endian integers:

| Field | Bytes |
| --- | --- |
| Version | 1 |
| Message type | 1 |
| Session token | 4 |
| Input or telemetry sequence | 4 |
| MCU timestamp in microseconds | 4 |
| Payload | 0–64 |
| CRC16-CCITT | 2 |

CRC uses initial value `0xffff` and polynomial `0x1021`, covering header and
payload. COBS encodes the packet, and a zero byte delimits it. CRC detects
corruption; it does not authenticate the peer.

Session and freshness handling matter independently of framing. Linux must not
give old input a fresh timestamp or compare unsynchronized MCU and Linux clocks
as though they shared an epoch. Host pseudo-terminal tests and physical MCU/UART
tests validate different parts of this protocol.
