# AR8030 factory storage and calibration-offset selection

[Firmware](README.md) · [Configuration](ar8030-configuration.md) ·
[Power control](ar8030-power-control.md)

This study covers the matched Gnd485 18.21.10 read-only factory reader introduced
around trial 145 and preserved in the early incoming-power checkpoint. It
connects flash identification, partition selection, offset decoding and the
power planner. Trial 145 exercised **invalid-partition fallback**, not a
successful hardware factory-record read. Other boards and later P1 calibration
have separate evidence.

## Calibration rows are explicit inputs

The calibration loader looks up an exact JSON board-member name, including
names containing dots. An absent member permits `default`; a present malformed
member fails instead of silently selecting another board. At this checkpoint,
normal boot selected `default` pending a board-selector provider.

Rows cover requested levels 9–39, with four bytes per row: analog/digital for
each path. Analog values must fit six bits, digital values one byte. The parser
accepts the matched hexadecimal strings with a trailing comma but rejects
malformed or extra row elements. A valid-row bitmap distinguishes absent rows
from calibrated zero values. The matched external configuration supplies 28
rows, 9–36; it does not supply 37–39 simply because the command accepts those
level numbers.

Closed-loop paths use separately clamped requested-level-plus-signed-offset
rows. Open-loop paths require the unadjusted row they actually use. Validation
checks all required rows before publishing a plan. Offset resolution copies
calibration and preserves unrelated user, clock, role and policy state. The
matched chip-profile-1 integration uses offset positions 0 and 2 for its paths;
other band/path mappings require separate reconstruction.

## Factory record and fallback precedence

| Record bytes | Meaning |
| --- | --- |
| 0–1 | Little-endian version |
| 2–3 | Reserved, ignored by this decoder |
| 4–7 | Four signed offset bytes |

The decoder requires exactly eight bytes. Version 4 preserves all four signed
values. Version 3 changes only odd positions 1 and 3, and only signed values
9–15: subtract 12, producing −3…3. Other values remain unchanged. Unknown
versions leave the decoder output unchanged and let the caller choose fallback;
truncation is a malformed record, not a zero-offset record.

The resolver distinguishes four storage states:

| State | Selection |
| --- | --- |
| Unread | Return `-11`; do not invent a storage result |
| Unavailable | Valid JSON offsets, otherwise four zeros |
| Record present | Decode version 3/4; unknown version permits JSON/zero fallback |
| JSON-only | Explicitly skip storage and use JSON/zero fallback |

JSON `pwr_offset`, when present, must have exactly four integers in −6…6.
Malformed arrays fail atomically. This stricter native validation does not
reproduce original partial parsing/narrowing. The CSMA offset is retained
separately, defaulting to 8. Record decoding and fallback do not themselves
measure a board's RF power.

The retained original-instruction comparison mocked storage, JSON, logging and
copy helpers. Its 2,072 cases include every signed byte in each position for
both supported versions and the precedence matrix, with 2,052 directly decoded
valid records. The wiki review rebuilt the native resolver and reproduced all
selections. This verifies conversion/selection, not actual partition reads.

## Partition windows and bounds

The reader checks 4 KiB windows at flash offsets `0x8000` and `0x9000`. Their
layout includes a sector signature, an `EFI PART` header and 128-byte entries:

- Bytes 510–511 must be `55 aa`; the header starts at window offset 512.
- The matched header revision is `0x10000`, header size 92 and entry size 128.
- Entries begin at window offset 1,024; at most 24 fit in the checked window.
- Header CRC32 is calculated over 92 bytes with its CRC field zeroed. Entry
  CRC32 covers the declared entry count. These use reflected IEEE polynomial
  `0xedb88320`, initial all-ones and final complement.
- Partition LBAs are relative to the chosen window, not absolute flash zero.

Iteration retains the original prefix rule: a non-increasing LBA pair ends the
search, including a single-sector entry where first equals last. Native bounds
checks reject oversized addresses before multiplication/addition, duplicate
matching names and non-ASCII aliases of the requested name. It does not truncate
high LBA words into an apparently valid address.

A valid primary table is authoritative even if it lacks `factory`; that result
is `-2`, and the reader does not search the other table for a more convenient
answer. Invalid/read-failed table windows permit the second attempt. Once a
valid factory partition is selected, a failed record read does not select a
different table. The eight-byte record is published only after a complete read.
These choices matter when distinguishing unavailable storage from an unknown
record version and from malformed partition metadata.

## SPI transport and geometry discovery

The source transport uses controller base `0x01e00000` and clock control
`0x0a104008`. Injected register/time callbacks support polling tests. A transfer
is bounded by both 50,000 µs and 1,000,000 polls, preventing a frozen timer from
creating an infinite wait. Transport failure latches an error and clears ready
state; subsequent transfers do not silently continue from a failed controller.

Initialization waits for idle, rejects an asserted start bit before rewriting
configuration, and drains stale FIFO data. It disables DMA/receive-trigger use
and selects the matched single-lane, single-data-rate path. The original
2,000,000-rate request maps to source 0/divider 5 with clock gating around the
update; actual SCLK was not measured. The relevant register accesses are:

| Offset | Use in the read-only implementation |
| --- | --- |
| `0x10` | Start and framing controls |
| `0x14` | Opcode, address-byte and dummy-byte controls |
| `0x18`, `0x1c` | Address scalar and requested byte count |
| `0x30` | Busy bit |
| `0x34` | FIFO availability in low seven bits |
| `0x100` | FIFO data word, drained low byte first including partial tails |

ID reads use opcode `0x9f` for six bytes; array reads use `0x03` and three address
bytes. SFDP uses `0x5a`, three address bytes and one dummy byte. The address
register receives the original controller's little-endian scalar convention;
it is not a software-swapped serial byte stream. No write/erase commands are
implemented by this reader.

Identification first checks 19 matched original ID-prefix/read-bound records
from `0x264338`. The generated profile contains those facts, not original code
or name pointers. Entry 18 duplicates entry 16's prefix with a different
capacity; first match deliberately retains the original 2 MiB bound. These are
supported-part bounds, not independent physical capacity measurements.

Unknown IDs require valid SFDP rather than assuming the original generic 2 MiB
fallback. The native parser requires a major-1 SFDP header and an initial
major-1 basic parameter header, then chooses the highest supported minor
version, breaking ties by table length. It reads the mandatory first nine
words, checks table bounds, and rejects four-byte-only/reserved addressing.
Density is decoded with checked 64-bit arithmetic. Physical size is retained
separately from the three-byte readable window, capped at 16 MiB. A failed
identification clears previously known capacity; zero capacity permits ID
commands but refuses array reads. Floating IDs beginning `00`/`ff` are rejected.

## Normal boot and the observed fallback

Factory preparation owns one identification/partition attempt after chip/OTP
discovery and before RF pin/default/calibration setup. Its physical binding
masks the original SPI CLIC interrupt, uses a static 4 KiB scratch buffer and
the existing coherent 40 MHz timer. Unsupported identification/controller
state stops boot. Actual partition/read failure permits the configured
JSON/default fallback.

Trial 145 observed `c46015c46015`, selected the 2 MiB bound, and returned `-74`
for both partition windows. Default path offsets `[0,0]` reached initial power
planning; restored stock's `0x01000071` getter also returned `[0,0]`. That getter
accepts four bytes, band plus reserved, and returns two signed offsets plus
reserved bytes. Reserved bytes are not extra calibration values.

Boot diagnostic version 2 has eighteen LE32 words. The appended words 8–12
contain factory stage, signed error, capacity, partition offset and offset
source; 13–14 contain signed path offsets; 15–16 retain six observed ID bytes
and the complete-ID flag; word 17 identifies the geometry source. These are
native diagnostics, not a vendor host ABI.

[Trial 145's video and recovery record](../experiments/ar8030-recovery-and-power.md#factory-fallback-with-working-video-trial-145)
proves boot/video with fallback. It does not establish why native partition
validation failed, what stock read from storage, successful real factory reads,
cold-start SPI pin setup or calibrated RF output. The partition/SPI/SFDP tests
used synthetic IO and an independent zlib CRC oracle; they were not hardware
flash qualification. For this wiki review, source was inspected and the
factory-selection comparison rerun; SPI/partition hardware was not exercised.
