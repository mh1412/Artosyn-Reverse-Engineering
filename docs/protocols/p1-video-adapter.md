# P1 UDP records and Ascent video adaptation

[Protocols](README.md) · [Interoperability experiment](../experiments/p1-ascent-interoperability.md)

The October 22 P1 1.0.44.rel prototype retained the stock camera pipeline and
adapted its compressed records for Ascent Gnd485 18.21.10. It changed framing
and removed selected metadata NALs without re-encoding picture data.

## Native P1 records

UDP port 10000 carries media requests; port 10001 carries video. The tested
zero-body requests are LE32 command ID followed by four zero bytes, using IDs
1/3/4/5. The adapter repeats IDs 1, mode 4 or 5, and 3 every two seconds during
streaming. These observations do not constitute the entire control protocol.

A video record has nine LE32 words, HEVC bytes and a four-byte tail:

| Offset | Field used by the examined adapter |
| --- | --- |
| 0 | Magic `0x12345678`, wire `78 56 34 12` |
| 4 | HEVC payload byte length `n` |
| 8 | Channel, 0 or 1 |
| 12 | IDR-related indicator; historical keyframe mode selects value 1 |
| 16 | Source frame ID |
| 20 | Source timestamp in milliseconds |
| 24 | Packed resolution: width in high 16 bits, height in low 16 bits |
| 28 | Header marker `0x87654321` |
| 32 | Additional header word; not verified by these prototype parsers |
| 36 | HEVC payload |
| `36+n` | Tail `0x87654321`, wire `21 43 65 87` |

Total record size is `n + 40`. Records can span UDP datagrams, including the
observed 32,768-byte split, so one datagram is not one picture. The examined
Python/C parsers accumulate data, enforce a 1 MiB record limit, validate channel
and markers, and resynchronize bytewise. They do **not** verify the word at
offset 32. A related reference pipeline describes a header CRC; its presence
must not be confused with checksum validation by this P1 prototype.

Native split mode produces independent 1920×560 and 1920×552 HEVC streams with
32 rows of overlap. The channels must be decoded separately and matched in time
before reconstructing a full picture. Early recovered frames from different
timestamps were not synchronized full-frame evidence. Later continuous split
and full 1920×1080 modes were both validated in named runs.

The full-frame adapter discards initial split records while the mode request
takes effect, requiring channel 0 and packed resolution `(1920 << 16) | 1080`.
This is a fixed-mode prototype; it does not negotiate arbitrary sizes.

## Native Ascent wrapper produced by the adapter

The adapter emits 46 header bytes, filtered HEVC and four trailer bytes:

| Offset | Bytes | Value |
| --- | ---: | --- |
| 0–4 | 5 | `00 00 00 01 80` |
| 5 | 1 | Sum of header bytes 6–45 modulo 256 |
| 7 | 1 | P1 IDR-related word converted to 0/1 |
| 8 | 4 | LE32 filtered payload length |
| 13 | 1 | Fixed nominal FPS 60 |
| 16 | 2 | LE16 adapter sequence, wrapping independently of source ID |
| 18 / 20 | 2 each | LE16 width 1920 / height 1080 |
| 22 | 8 | LE64 P1 millisecond timestamp multiplied by 1,000 |
| 30 | 4 | Original LE32 millisecond timestamp |
| Other header bytes | — | Zero in this adapter |
| 46 | Variable | HEVC after SEI filtering |
| End | 4 | `ee 29 55 9f` |

Total output size is filtered payload plus 50 bytes. Multiplying the same source
timestamp into another field does not measure capture-to-completion delay or
establish the full semantics of stock Ascent timestamps.

Annex-B parsing accepts three- or four-byte start codes and derives HEVC NAL type
as `(first_header_byte >> 1) & 63`. Types 39/40 are omitted; all other NAL spans
are copied unchanged. Input must start with a start code and contain a complete
two-byte NAL header. This avoids the observed native Ascent userdata-buffer
failure while preserving decoded pixels in the tested sample. The filter is
not a claim of compatibility with every HEVC metadata use.

## Daemon transport and queue completion

The on-P1 adapter writes through the stock daemon's loopback TCP service. A
separate data connection starts with OPEN, selecting TX and 32,768-byte capacity.
The daemon adds radio write positions and consumes completion replies; the
application sends chunks of at most 4,000 bytes and requires the matching byte
count with an empty reply body. Native Ascent mode selects SDK port 3, already
owned by the receiver media backend; independent probing must not take that
receiver port away from it. Raw host-capture modes use port 0 instead.

Before closing, the adapter polls native per-port diagnostics for queued bytes
and active writes to drain, with a five-second bound, then leaves a short settling
interval. These local counters do not prove remote decode or display completion.
The source-record byte count is measured **before** reframing/filtering, so it is
not the same as Ascent wire bytes.

An older low-rate experiment used repeated independent blocks:

| Offset | LE32 field, except magic |
| --- | --- |
| 0 | ASCII `P1RF` |
| 4 | Adapter frame number |
| 8 | Complete native record length |
| 12 | Byte offset in that record |
| 16 | Block payload length, at most 512 |
| 20 | FNV-1a over payload, initial `2166136261`, multiplier `16777619`, modulo 2³² |
| 24 | Payload |

It sent eight passes with 5 ms inter-block pauses to recover isolated exact
records despite missing stream bytes. This is historical transport evidence,
not the final native-HDMI protocol or an efficient low-latency design. FNV-1a
detects accidental changes in that experiment; it is not authentication.
