# Sky482 NAND capture and controller page layout

[Firmware](README.md) · [Ascent Lite](../devices/ascent-lite-vtx.md) ·
[Repair experiment](../experiments/ascent-lite-flash-repair.md)

The October 16, 2026 Ascent Lite repair recovered both chip-visible NAND bytes
and the controller-visible filesystem layout. Those are different representations.
The observations below apply to board `FPV-Ascent-Sky-482-V1.3-1.1` with stock
18.21.10 and the particular flash capture used for repair.

## Flash identity and capture geometry

The owner-reported marking was interpreted as W25N01GVZEIG. Electronic ID
`ef aa 21` confirmed the W25N01GV family; the package suffix comes from the
marking. The owner reported the chip isolated from the SoC, measured 3.3 V,
and added separate /HOLD and /WP pull-ups. Consistent identification followed
those pull-ups. This does not provide a verified VTX test-pad/isolation map.

| Quantity | Recorded value |
| --- | ---: |
| Main bytes per page | 2,048 |
| Spare bytes per page | 64 |
| Pages per erase block | 64 |
| Total pages / blocks | 65,536 / 1,024 |
| Main capacity | 128 MiB |
| Page-plus-spare capture | 138,412,032 bytes, or 132 MiB |
| Initial configuration / protection | `0x18` / `0x7c` |
| Block-remapping table | 80 zero bytes; no active entries in the observed capture |

The reader preserves spare bytes and does not skip bad blocks. Its addresses
remain **chip-visible**: internal remapping is recorded, not bypassed. A zero
table in this unit does not make every such capture a guaranteed unremapped
physical array image.

## Reader interface and raw-mode boundary

The XIAO RP2350 used SPI mode 0 and the following wiring after isolation:

| XIAO label | GPIO | Flash package pad | Signal |
| --- | ---: | ---: | --- |
| D3 | 5 | 1 | /CS |
| D9 | 4 | 2 | MISO / DO |
| D10 | 3 | 5 | MOSI / DI |
| D8 | 2 | 6 | CLK |
| GND | — | 4 | Common ground |

Target power was independently supplied; this is not an instruction to connect
the programmer's power rail to a populated VTX. The operator's bus-isolation
assertion is not an electrical measurement performed by software.

The ordinary reader implements ID/register/remapping-table reads, page-to-cache
read and cache read. It has no array-program, erase, write-enable, arbitrary SPI,
OTP-write or remapping-write interface. Explicit raw mode adds only the constrained
volatile configuration command `1f b0`: save the initial `0x18` or `0x08`, clear
ECC to `0x08`, then restore the original setting and verify protection unchanged.

Release, idle timeout, disconnect and error paths attempt restoration before
releasing the pins; failure latches a fault that prevents re-arming. Cleanup
returns to 500 kHz even after a faster read. Loss of programmer power can prevent
restoration. A firmware release acknowledgement is not independent measurement
of high-impedance pins through every reset state; the capture metadata explicitly
left `reader_release_hardware_validated` false.

Reader host protocol v1 uses line commands `INFO`, `ARM ISOLATED`, `PROBE`,
`RAW BEGIN`, `PAGE n`, `RELEASE` and, from version 1.4, `SPEED hz` while released.
Most responses are JSON lines. A page response has a 16-byte little-endian
`<4sIBBHI` header:

| Offset | Bytes | Field |
| --- | ---: | --- |
| 0 | 4 | `NAND` magic |
| 4 | 4 | Page number |
| 8 | 1 | Status register |
| 9 | 1 | Configuration register |
| 10 | 2 | Payload length, 2,112 |
| 12 | 4 | CRC32 of header bytes 4–11 followed by payload |
| 16 | 2,112 | Complete page plus spare bytes |

Transport corruption or truncation stops capture without padding or silent
retry. The initial firmware mixed buffered newlib output with Pico USB output,
leaving part of a response buffered; version 1.2 switched to the raw Pico path.
Version 1.3 introduced raw sessions, 1.4 speed selection, and 1.5 requests up to
16 MHz. A transport CRC detects transfer damage, not incorrect sampling at the
flash pins; slower-baseline comparisons were a separate check.

## Built-in ECC and repeatability

Initial ECC-enabled samples reported uncorrectable status on every page.
Raw versus ECC-on samples differed in 171 bytes. The SoC's stored ECC/layout
therefore could not be treated as compatible with the chip's correction scheme.
The ECC-on result was unsuitable as the uncorrected baseline.

Two raw 64-page samples matched, including spare bytes. Five 64-page regions
at requested 2/4 MHz matched their 500 kHz baselines. Later five 256-page regions
at requested 8/16 MHz matched the first full capture; actual rates were 7.5/15 MHz.

| Complete capture | Actual SPI clock | Duration | Result |
| --- | ---: | ---: | --- |
| First original | 3,947,368 Hz | 615.433 s | 65,536 pages, restoration verified |
| Second original | 15,000,000 Hz | 368.588 s | Exact full-image match, restoration verified |
| Post-repair | 15,000,000 Hz | 348.170 s | Exact candidate match, restoration verified |

The earlier 500 kHz run was interrupted after 8,015 complete pages. Its metadata
retained an unverified cleanup result because the interrupted binary response
obscured the acknowledgement; a subsequent independent probe confirmed restored
configuration/protection. Its entire saved prefix matched the first complete read.

With ECC disabled, zero ECC-status bits do not establish error-free cells.
Repeated full-image agreement establishes stable readback under these conditions,
not universal flash health or compatibility with an arbitrary writer.

## Stored partition map

The environment and later live command line agree on this main-area layout:

| Partition | Main-data offset | Size |
| --- | --- | ---: |
| boot | `0x00000000` | 512 KiB |
| env | `0x00080000` | 512 KiB |
| sel | `0x00100000` | 1 MiB |
| kernel0 | `0x00200000` | 4 MiB |
| kernel1 | `0x00600000` | 4 MiB |
| factory | `0x00a00000` | 6 MiB |
| rootfs0 | `0x01000000` | 16 MiB |
| rootfs1 | `0x02000000` | 16 MiB |
| usrdata0 | `0x03000000` | 32 MiB |
| usrdata1 | `0x05000000` | 32 MiB |

These partitions cover 112 MiB; the complete capture preserves the remaining
16 MiB too. Main-data offsets are not byte offsets into 2,112-byte capture
records. For a page-aligned main offset `a`, its capture-record start is
`(a / 2048) × 2112`; finding controller data within that page also requires the
mapping below.

Stored settings included `bootcmd=fpvboot`, `ubi.mtd=6`, `fpvbsel=0` and
`console=ttyAMA0,115200`. Stored settings alone did not prove the selected boot
bank; the recovered runtime later confirmed bank 0. UBI erase-counter headers
reported VID-header offset 2,048 and data offset 4,096, with valid header CRCs.
An erased VID header in a sampled first block did not imply an absent filesystem.

## Reconstructed controller layout

Simply keeping the first 2,048 bytes of each record produced extraction errors.
The validated mapping for the affected root/application pages is:

```text
controller_data = physical[0:1040]
                + physical[1054:2048]
                + physical[2050:2064]
```

Ranges use an exclusive end. Their lengths are 1,040 + 994 + 14 = 2,048.

| Physical range | Bytes | Observed contents |
| --- | ---: | --- |
| `[0:1040]` | 1,040 | Controller data `[0:1040]` |
| `[1040:1054]` | 14 | First masked BCH parity |
| `[1054:2048]` | 994 | Controller data `[1040:2034]` |
| `[2048:2050]` | 2 | `ff ff` in the reconstructed layout |
| `[2050:2064]` | 14 | Controller data `[2034:2048]` |
| `[2064:2078]` | 14 | Second masked BCH parity |
| `[2078:2112]` | 34 | Fixed observed tail |

The encoder uses BCH strength 8, polynomial `0x4443`, with bit swapping enabled
(`bchlib` 2.1.3). Its exact observed constants are:

```text
mask1 = 0a3ae93943de09ac8322d0e17ff3
mask2 = 895c89101b7bb5679e67ba9be219
tail  = ffffffffffffffffffffffffffffffffffffffffffffffffffffffff0000ffffffff
```

Parity 1 is `BCH(data[0:1040]) XOR mask1`. Parity 2 is
`BCH(data[1040:2048] + tail[0:32]) XOR mask2`. Both BCH inputs are 1,040 bytes.
The fixed 34-byte tail occupies the last range above. These are empirically
validated image-specific constants, not a general HiSilicon NAND specification.

A fully erased 2,112-byte page remains all `ff`. A programmed page whose decoded
main data happen to be all `ff` must instead retain its parity/tail. The validation
round trip distinguishes these cases by inspecting the complete original record.
It reproduced **all 24,576 original pages in rootfs0 and usrdata0 exactly**.
Applying the decode mapping also produced valid checksums for scanned UBIFS nodes
in both application volumes.

The layout decoder only rearranges bytes; it does not correct bit errors. Its
validator requires exact reconstruction and rejects unfamiliar markers/parity.
Analysis images with relocated master nodes were extraction aids, never write
candidates. The subsequent repair used Linux UBIFS journal replay in a NAND
simulator and preserved a separately validated chip-visible candidate.
