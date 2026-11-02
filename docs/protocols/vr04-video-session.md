# VR04 reference media sessions and tiled HEVC records

[Protocols](README.md) · [SDIO transport](vr04-sdio-transport.md) ·
[Link control](../systems/vr04-link-control.md)

## Scope and endpoints

This is the NT99235/VR04-related reference protocol, including the retained
vendor-receiver interoperability findings and MissingLynk userspace revision
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
air repeating its identity. The open air role mirrors the received probe with
byte 0 changed to `1`. Hello probing stops once the exchange completes.

On UDP 10000, type 1 requests media parameters, type 2 replies, and type 3
requests an IDR. The vendor TX handler starts its pipeline or forces an IDR
when it receives type 3. The reference stream normally begins with an IDR per
tile and then P-frames; there is no periodic keyframe guarantee. Later receivers
must request an entry point instead of waiting indefinitely.

The open receiver delays its first media request until a consumer sends READY
on its local link socket. READY expires after six seconds. `frames_seen` means
the consumer's composed-frame report, not simply receipt of UDP data. Type-3
requests continue on parameter replies until that report confirms video; the
air limits forced keyframes to one per 500 ms.

The selected implementation sends type-1 requests every two seconds for the
**whole session**. After parameters are acknowledged this is an ungated keepalive,
even if READY subsequently expires. Earlier notes say polling stops after video
starts; that is not the later open daemon's behavior. Stock traffic observations
also include long quiet periods, so open keepalive policy must not be generalized
to every vendor receiver.

More than five seconds without receiver-side port-10000 traffic marks air loss
and clears handshake state while baseband polling continues. A separate six-second
stall of the consumer's video-datagram counter, after video had been confirmed,
restarts media negotiation. A stale cumulative `frames_seen` report can still
prematurely confirm a new session and suppress IDR requests; it does **not** stop
the later daemon's continuing type-1 keepalive.

### What the open handlers actually acknowledge

The expected 520-byte identity exchange above is not an enforced receive length.
The air accepts any positive-length hello beginning with byte 0; the receiver
accepts any positive-length identity beginning with byte 1. Both ignore the
sender address returned by the socket API and send to their configured peer
address. The receiver marks the identity exchange done after attempting its
ACK, without checking that send's result.

Similarly, the receiver's parameter handler accepts a four-byte type-2 value
without checking the declared body length or readiness fields. It attempts
SetLdCfg when configured, then IDR, and marks parameters acknowledged regardless
of send errors. Subsequent replies retry IDR until video is confirmed, but do
not retry that session's failed SetLdCfg. This permissive open handler cannot
serve as a test of the stricter vendor parameter format described below.

Live camera and scale commands clear their pending flags before `sendto`.
A failed send is retried only after another command or session re-arms it.
The two-second power/standby assertion has a separate periodic retry policy.
Its accepted power choices are 25/100/200 mW labels mapped to 14/20/23 dBm
control bytes; those labels are not new radiated-power measurements.

Receiver liveness refreshes on **any positive-length** port-10000 datagram,
before message parsing, including unknown or too-short messages. The five-second
loss watch requires STEADY and a previous receive timestamp. Air-side power
liveness instead requires at least four bytes, but also refreshes before checking
whether the type has a handler. Neither is proof that a valid media session is
making progress.

READY's packet counter is tested for inequality, so a decrease or reset refreshes
the progress time too. Its nonzero cumulative frame count can confirm video
without demonstrating a fresh frame in this session. The stall watch does not
require currently fresh READY. If it clears `params_acked` after READY has
expired, media requests become READY-gated again; the ongoing keepalive is not
unconditionally maintained through that restart.

The air bounds each receive drain to 32 datagrams per socket per tick. The
receiver drains each socket until empty, so its nominal 50 ms tick does not
bound service time under continuous input. Send cadence and timeout checks use
one sampled loop time; they are not independent deadline workers.

Offline probes of the unmodified receiver handlers, with all sends forced to
fail, reproduced four-byte parameter acceptance, one-shot configuration loss,
camera/scale pending-flag consumption and cumulative/counter-reset READY
behavior under AddressSanitizer and UndefinedBehaviorSanitizer. This checks
source contracts, not packet loss rates or live peer interoperability.

The air parameter reply advertises fixed 1920×1080 at 60 fps and ready fields;
it does not query the separate video process. Replying therefore does not prove
that its encoder is running or agrees with the advertised configuration. The
500 ms IDR limiter advances only after the local `keyframe` control send
succeeds. Failed local sends remain eligible for a later request, while a
successful control send still does not independently verify an emitted IDR.

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
| 44 | float32 | Captured 59.99988; open builder uses exact 60.0 |
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

The retained host test compares every byte against an embedded captured reply,
except the explicitly allowed floating-point frame-rate difference. That check
was rerun successfully during consolidation. It verifies the builder against
the retained fixture, not a new vendor-device exchange or an independent replay
of the original packet capture. The open receiver ignores most of this body,
so successful open-to-open video cannot test vendor acceptance of these fields.

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

The retained packetizer check also passed during consolidation: the standard
CRC vector `123456789 → cbf43926`, an independently generated complete packet,
and an embedded vendor SEI sample matched. Its small synthetic NAL tests check
SEI placement, not general HEVC acceptance or live radio delivery.

The selected open sender places one complete record in one UDP datagram, capped
at 65,507 bytes: at most **65,467 payload bytes including inserted SEI**.
IP fragments datagrams above the 4,096-byte link MTU. Oversized access units
are dropped rather than split or truncated; oversized initial IDRs trigger
bounded requests for a smaller keyframe. With no periodic IDR, exhausting that
recovery can leave a tile undecodable. Encoder bitrate/QP affects whether an
entry point fits, but those settings alone do not guarantee a fit for every scene.

This differs from the separate stock P1 prototype's observed 32,768-byte
record splitting across UDP datagrams. Reuse the common header knowledge without
assuming identical packetization. A successful nonblocking `sendto` and local
send counter establish kernel acceptance, not RF delivery or display.

The open receiver does not enforce every sender field: it ignores the payload
tail marker and header IDR/timestamp/resolution, and its length addition can
wrap before comparison. Its startup gate also accepts VPS alone. The
[receive implementation review](../systems/vr04-frame-ownership.md#datagram-admission-and-decoder-startup)
documents these limits and the separate SEI telemetry parser; accepting an open
receiver input is not evidence that it satisfies the vendor requirements below.

## Vendor HEVC acceptance requirements

The recovered receiver performs a positional IDR-header check, not just a
general Annex-B parse:

| Offset | Required bytes |
| --- | --- |
| 0–5 | `00 00 00 01 40 01`, VPS first |
| 39–44 | `00 00 00 01 42 01`, SPS at the expected position |

Thus the first VPS NAL occupies 35 bytes excluding its four-byte start code.
A three-byte start code or an AUD placed first fails this check. The open
encoder's former seven-byte AUD caused repeated IDR requests and a black panel
despite decoding successfully on the open receiver. Its AUD control is disabled
for this vendor-compatible path.

Each access unit also needs a prefix SEI carrying unregistered user data.
Without it, the vendor accepts IDRs and then fails `AR_MPI_VDEC_GetUserData`,
tearing down reception. The earlier “diagnostic only” interpretation is refuted
by that recorded failure. Insert the SEI immediately before the first VCL NAL:
after VPS/SPS/PPS for IDR, before the slice for P-frames.

The recovered UUID is `bde945dc-b748-d9e6-20d8-2c96efee23d9`. NAL type 39
uses payload type 5 and a variable length covering the UUID plus NUL-terminated
text: `ChnId %d FrameId %d PTS %x Filed %d BR %d QP %x`.
`Filed` is the original spelling and was 4 in sampled frames. PTS/QP are hex;
bitrate is decimal kbps. The open sender supplies PTS as milliseconds ×1,000
and configured QP, since it has no per-frame measured QP. Whether the receiver
interprets the text fields or only requires valid user data remains unresolved.

The recorded vendor VPS declares Level 4.0 (`general_level_idc=120`). An open
capture unexpectedly declared 10, despite the control's nominal default mapping;
the sender explicitly requests Level 4.0. Actual emitted parameters need checking:
an accepted control value is not evidence of the serialized value.

## Session clocks and measurement limits

A receiver reboot resets its time origin. A continuing transmitter timestamp
caused a measured 31.38-second future skew: decoder counters and initial display
messages looked healthy, while the presentation stage discarded frames.

The later daemon requests timestamp rebasing on the first identity hello of an
episode, separated by a five-second hello gap. Parameter requests are keepalives
and no longer reset the session. This supersedes the older type-1 reset trigger.
The air uses a strict `gap > 5000 ms` test and advances its last-hello time even
when sending `session-reset` to the video process fails. Thus a failed reset is
not retried by the next closely spaced hello. The initial last-hello value is
zero, so a first probe within five seconds of the monotonic clock origin does
not request a reset. The identity reply is still attempted.
The transmitter queues per-tile reset flags and applies them in the access-unit
writer; direct mutation by the control thread had broken tile pairing.
Frame IDs continue across sessions. The vendor warned but accepted the later
IDRs in three reported reassociations; this is not a general shared-frame-ID proof.

Live packet sniffing avoids spending a camera bring-up merely to inspect headers.
The first IP fragment suffices for header/size checks; reconstructing elementary
streams requires all fragments and an IDR in the capture window. Recorded capture
socket losses of 51–72 among roughly 400–550 datagrams mean clean decoded samples
do not establish loss-free RF delivery. See the
[camera validation](../experiments/nt99235-camera-validation.md) for stale-memory
and camera-restart confounders below the packet layer.
