# P1 stock UDP video records

[Protocols](README.md) · [P1 Air](../devices/betafpv-p1-air.md)

## Captured stock P1 records

UDP port 10000 carries media requests; port 10001 carries video. The tested
zero-body requests are LE32 command ID followed by four zero bytes, using IDs
1/3/4/5. These observations do not constitute the entire control protocol.

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
offset 32. The [related reference pipeline](vr04-video-session.md) describes a
header CRC; its presence
must not be confused with checksum validation by this P1 prototype.

Native split mode produces independent 1920×560 and 1920×552 HEVC streams with
32 rows of overlap. The channels must be decoded separately and matched in time
before reconstructing a full picture. Early recovered frames from different
timestamps were not synchronized full-frame evidence. Later continuous split
and full 1920×1080 modes were both validated in named runs.
