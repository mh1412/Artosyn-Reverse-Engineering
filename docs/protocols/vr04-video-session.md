# VR04 reference media sessions and tiled HEVC records

[Protocols](README.md) · [SDIO transport](vr04-sdio-transport.md)

## Scope and endpoints

This is the NT99235/VR04-related reference protocol, using retained
vendor-receiver findings from research archived at userspace revision
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`. It is an application layer over an
AR8030 IP tunnel, not the RF modulation or wireless MAC format. The later
[SC231AI P1 adapter](p1-video-adapter.md) has separate packetization observations.

| Endpoint | Role |
| --- | --- |
| Receiver `10.0.0.1`, air `10.0.0.100` | Reference `sdio0` addresses |
| UDP 20001 | Identity hello/acknowledgement |
| UDP 10000 | Media parameters, keyframe requests, configuration, telemetry and OSD |
| UDP 10001 | Framed HEVC access units; one captured air source port was 49958 |
| `/dev/artosyn_sdio` | Separate local baseband control interface |

An associated radio does not necessarily transmit video. The recorded working
sequence includes baseband association/polling, receiver identity and a media
handshake with a listening consumer.

## Establishment and recovery

The identity exchange starts with a 520-byte type-0 probe, approximately three
times per second. The air replies with type 1. The receiver echoes that packet
with byte 0 changed to `2` and byte 5 to `0`; failing to acknowledge leaves the
air repeating its identity. Hello probing stops once the exchange completes.

On UDP 10000, type 1 requests media parameters, type 2 replies, and type 3
requests an IDR. The vendor TX handler starts its pipeline or forces an IDR
when it receives type 3. The reference stream normally begins with an IDR per
tile and then P-frames; there is no periodic keyframe guarantee. Later receivers
must request an entry point instead of waiting indefinitely.

## Media header and the 104-byte parameter reply

Scalar fields below are little-endian. The common fields occupy bytes 0–19:
type at 0, a 32-bit microsecond timestamp at 8, and body length at 16. Bodyless
requests occupy 24 bytes with declared length zero. This does not put every
message's body at byte 24: the parameter/configuration bodies start at **20**.

The actual parameter builder emits **104 bytes = 20 + 72 + 12**. This corrects
both the older 92-byte description and the misleading “24-byte header + body”
arithmetic. The declared body length is `0x48`; the final 12 bytes are zero.

| Datagram offset | Type | Reference value and meaning |
| --- | --- | --- |
| 0 | u32 | `2`, media parameters |
| 8 | u32 | Message timestamp |
| 16 | u32 | `0x48`, required declared body length |
| 20 | u32 | `1`, use the tile pattern's decoder-channel count |
| 24 | u32 | `1`, pipeline value |
| 28 | u32 | `1`, source ready |
| 32 | u32 | `0` on the captured wire, despite a transient producer path setting 1 |
| 36 / 40 | u32 | Both zero; narrative labels CSC/interlace are not corroborated by the builder's still-unknown annotations |
| 44 | float32 | Captured 59.99988 |
| 48 / 52 | u32 | Composite width/height, 1920/1080 |
| 56 | u32 | Whole frame rate, 60 |
| 60–87 | bytes | Zero, remaining semantics not established here |
| 88 | float32 | `1.0` |
| 92–103 | bytes | Twelve zero trailer bytes |

The vendor first checks declared length, then readiness. A zero flag at offset
20 creates only decoder channel 0 even when the stream contains two tiles.
Channel 1 then rejects data with `bs_size = 0` and `0x80058403`; this is a missing
decoder-channel condition, not proof of malformed HEVC. Repeated rejections can
reset the receive pipeline permanently for that boot.

## Video record layout and transport size

| Offset | LE32 field |
| --- | --- |
| 0 | Magic `0x12345678` |
| 4 | HEVC access-unit length `n` |
| 8 | Channel/tile index |
| 12 | IDR indicator |
| 16 | Frame ID |
| 20 | `TimeStap`, milliseconds |
| 24 | Composite resolution: `(width << 16) | height` |
| 28 | Header marker `0x87654321` |
| 32 | CRC-32 over bytes 0–31 only |
| 36 | HEVC access unit |
| `36+n` | Tail marker `0x87654321` |

CRC uses reflected polynomial `0xedb88320`, initial/final XOR `0xffffffff`.
It does not cover HEVC bytes and is not authentication. Both tiles advertise the
**composite** 1920×1080 resolution. Channel 0 carries 1920×560 and channel 1
1920×552; their 32-row overlap joins top rows 528–559 to bottom rows 0–31.
They are independent bitstreams requiring time/frame matching and seam composition.

The [separate stock P1 capture](p1-video-adapter.md) observed records split
across datagrams. Shared header fields do not establish identical packetization.

## Vendor HEVC acceptance requirements

The recovered receiver performs a positional IDR-header check, not just a
general Annex-B parse:

| Offset | Required bytes |
| --- | --- |
| 0–5 | `00 00 00 01 40 01`, VPS first |
| 39–44 | `00 00 00 01 42 01`, SPS at the expected position |

Thus the first VPS NAL occupies 35 bytes excluding its four-byte start code.
A three-byte start code or an AUD placed first fails this check. A test stream with a seven-byte AUD before the VPS caused repeated IDR requests
and a black panel despite being decodable by a general-purpose receiver.

Each access unit also needs a prefix SEI carrying unregistered user data.
Without it, the vendor accepts IDRs and then fails `AR_MPI_VDEC_GetUserData`,
tearing down reception. The earlier “diagnostic only” interpretation is refuted
by that recorded failure. Insert the SEI immediately before the first VCL NAL:
after VPS/SPS/PPS for IDR, before the slice for P-frames.

The recovered UUID is `bde945dc-b748-d9e6-20d8-2c96efee23d9`. NAL type 39
uses payload type 5 and a variable length covering the UUID plus NUL-terminated
text: `ChnId %d FrameId %d PTS %x Filed %d BR %d QP %x`.
`Filed` is the original spelling and was 4 in sampled frames. PTS/QP are hex;
bitrate is decimal kbps. Whether the receiver
interprets the text fields or only requires valid user data remains unresolved.

The recorded vendor VPS declares Level 4.0 (`general_level_idc=120`).

## Session clocks and measurement limits

A receiver reboot resets its time origin. A continuing transmitter timestamp
caused a measured 31.38-second future skew: decoder counters and initial display
messages looked healthy, while the presentation stage discarded frames.

Live packet sniffing avoids spending a camera bring-up merely to inspect headers.
The first IP fragment suffices for header/size checks; reconstructing elementary
streams requires all fragments and an IDR in the capture window. Recorded capture
socket losses of 51–72 among roughly 400–550 datagrams mean clean decoded samples
do not establish loss-free RF delivery. See the
[camera validation](../experiments/nt99235-camera-validation.md) for stale-memory
and camera-restart confounders below the packet layer.
