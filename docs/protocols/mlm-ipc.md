# MissingLynk local messaging and DRM ownership

[Protocols](README.md) · [Open reference media](../systems/vr04-open-media.md)

## Scope and framing

MLM is the open reference runtime's local process protocol, not the Artosyn RF
wire protocol or the stock MPP RPC ABI. This describes the inspected shared
header and DRM broker at userspace revision
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`.

All sockets live under `/run/missinglynk`. Except for `drm.sock`, they use Unix
datagrams: one datagram carries an eight-byte packed header and its payload.

| Offset | Header field |
| --- | --- |
| 0 | `u32` magic `0x314d4c4d`, bytes `MLM1` on the reference little-endian target |
| 4 | `u16` record type |
| 6 | `u16` flags |

There is no header length, sequence or checksum. Payload length comes from the
datagram. Structures are packed and written directly in native byte order; this
is not a portable cross-endian network serialization. Enum-valued fields are
explicit-width integers, allowing unknown appended values to reach consumers'
default handling. Producer comments promise nonblocking publication, but the
inspected convenience command senders create ordinary sockets and call `sendto`
with flags 0. Nonblocking/drop-on-error behavior must be checked per sender,
not assumed from the protocol's overview.

| Socket | Binder and purpose |
| --- | --- |
| `drm.sock` | DRM broker; stream connection passes the shared display fd |
| `telemetry.sock` | HUD; receives radio status/link/scan and pipeline state/frame statistics |
| `osd.sock` | HUD; receives raw radio canvas frames |
| `link.sock` | Radio daemon; receives pipeline readiness and HUD RF commands |
| `ctrl.sock` | Pipeline; receives recording/playback/overlay commands |
| `led.sock` | LED daemon; receives desired pattern/color |

## Record types and payloads

Sizes below exclude the eight-byte MLM header and describe this snapshot.

| Type | Payload |
| --- | --- |
| `0x01` MSP | Variable raw UDP-10000 type-`0x10` frame, including its radio envelope |
| `0x02` status | Variable raw type-`0x09`/`0x11` radio frame |
| `0x03` frame statistics | 20 bytes: FrameId +0, combined bitrate kb/s +4, PTS ns `u64` +8, tile-0 QP +16 |
| `0x04` ready | 8 bytes: latched frames-seen flag and low 32 bits of raw video packet count |
| `0x05` link | 8 bytes: state and auxiliary value, including peer MAC for bind success |
| `0x06` LED | 8 bytes: mode/R/G/B, `u16` period ms and reserved field |
| `0x07` command | 8-byte command/argument pair, optionally followed by text or cell data |
| `0x08` pipeline state | 16 bytes: mode, flags, playback position ms and duration ms |
| `0x09` link information | 20 bytes: signed channel/SNR/distance, flags and throughput kb/s |
| `0x0a` RF command | 8-byte command/argument pair |
| `0x0b` scan | 160 bytes: eight-byte header and nineteen eight-byte channel entries |

Frame bitrate/QP fields come from encoder SEI interpretation; a configured or
reported QP does not independently measure actual per-block encoder decisions.
Radio throughput is separate from encoder bitrate. Link-information unknown
values use `INT32_MIN`, preserving valid negative SNR readings. The scan's signed
dB sentinel is `INT16_MIN`; raw SNR 0 means no channel lock and −1 means no reply.

The scan header contains valid bitmap, count, active index, measured flag and a
reserved byte. Entries contain MHz frequency, signed dB/raw SNR, index and valid
flag. A table-only seed has `measured=0`; an explicit retune-and-measure sweep
interrupts video and later restores the channel. The old RF-command comment
calling this “read-only” must not be interpreted as a passive sweep. The
[stock UI study](vr04-stock-ui.md) distinguishes raw table queries from retuning.

Link states 1–7 are associated, parameters acknowledged, transmitter lost,
session restart, binding, bind success and bind failure. Readiness/association
are not proof of a decoded visible picture; the
[link-control article](../systems/vr04-link-control.md) describes the later
keepalive, packet-stall and binding policies.

## Pipeline commands and state

| Command | Argument / trailing data |
| --- | --- |
| 1 record toggle | No additional payload |
| 2 play | NUL-terminated path, bounded to 512 bytes |
| 3 / 4 pause / resume | No additional payload |
| 5 seek | Position in thousandths of duration |
| 6 stop | Return to live operation |
| 7 speed | Signed multiplier carried as `u32` bits |
| 8 show idle | No-signal display request; ignored during playback |
| 9 SRT text | NUL-terminated telemetry text; meaningful while recording |
| 10 recording format | `(height << 16) | fps`, latched for the next recording |
| 11 OSD cell | Twelve-byte cell header followed by optional RGBA pixels |
| 12 RTSP | Idempotent on/off intent |
| 13 latency counter | Idempotent on/off intent for a counter burned into the composite |

Pipeline modes are idle/live 0, recording 1 and playback 2. Flags are paused
`0x01`, ended `0x02`, rendering `0x04`, live video `0x08`, RTSP `0x10` and latency
counter `0x20`. Ended remains playback with a pinned last frame. The rendering
signal is a software presentation milestone, not independent optical proof.
RTSP reports an enabled, running encoder, rather than merely accepting intent.
A counter feature intended for filming nested images supplies no latency result
until that experiment is actually measured.

The cell header contains six `u16` values: row, column, x, y, width and height.
The grid is 20×53 and patches are bounded to 64×64 RGBA pixels. A header without
pixels clears one cell; row and column both `0xffff` clear the cache. Alpha at
least 128 is opaque. The [open media study](../systems/vr04-open-media.md)
describes its conversion and recording semantics.

RF commands 1–9 cover standby, power, bitrate, channel, scan, camera, scale,
binding and fixed MCS. Power accepts 25/100/200 mW intent; those are converted
to radio command values, not measured output power. Camera packs selector/value
into the high/low 16 bits; scale packs aspect/zoom percentage. Binding argument
0 requests runtime-only pairing and 1 persistence; MCS `0xffff` means automatic.
Some header comments predate later standby and bitrate handling. The
[implemented link policy](../systems/vr04-link-control.md) is the authoritative
explanation of those changes, rather than treating the header comments as new
hardware verification.

## Shared DRM descriptor

The broker opens the display device once, attempts DRM master ownership, and
passes the same open file description with `SCM_RIGHTS` over a stream socket.
The one-byte ancillary-data carrier is `F`; it is not an MLM-framed datagram.
Clients close the control connection after receiving their descriptor.

Keeping the broker alive anchors the display session across a client restart.
Restarting it while old clients retain the previous open file description does
not transfer master ownership to its new descriptor. The broker logs an initial
master failure and still serves; receiving an fd alone does not prove modeset
authority. Sharing the descriptor also does not grant per-plane isolation.

The early broker comment prohibits DRM events because the event queue is shared.
The later pipeline does use page-flip events and `drmHandleEvent`, while the HUD
updates its overlay without becoming a competing event consumer. The enduring
constraint is ownership of that shared event queue, not an accurate claim that
the later pipeline uses only blocking `SetPlane`. Later HUD commit suppression
also addresses contention on the shared CRTC; see
[receiver timing](../experiments/vr04-receiver-latency.md).
