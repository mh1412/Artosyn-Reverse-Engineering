# AR8030 stock factory records and flash access

[Firmware](README.md) · [Configuration](ar8030-configuration.md) · [Power](ar8030-power-control.md)

This study retains the matched Gnd485 18.21.10 factory-record reconstruction.
Original-instruction comparisons used mocked storage, JSON and MMIO; they
establish conversion and selection behavior, not successful physical reads in
every experiment or calibrated RF output.

## Calibration record

| Record bytes | Meaning |
| --- | --- |
| 0–1 | Little-endian version |
| 2–3 | Reserved, ignored by this decoder |
| 4–7 | Four signed offset bytes |

Version 4 preserves all four signed
values. Version 3 changes only odd positions 1 and 3, and only signed values
9–15: subtract 12, producing −3…3. Other values remain unchanged. Unknown
versions leave the decoder output unchanged and let the caller choose fallback;
truncation is a malformed record, not a zero-offset record.

Recognized factory records take precedence over JSON offsets; unavailable or
unrecognized records can fall back to JSON and then zeros. A failed read must
not be reported as a measured zero-offset record. The 2,072 retained comparison
cases include every signed byte at each position for both supported versions.

## Partition and SPI findings

The studied partition reader examines 4 KiB windows at offsets `0x8000` and
`0x9000`. Their format includes signature `55 aa` at bytes 510–511, an `EFI PART`
header at byte 512, revision `0x10000`, 92-byte header and 128-byte entries at
byte 1024. Partition LBAs are relative to the selected window. The original
iteration stops at a non-increasing LBA pair, including first equal to last.

The SPI controller is at `0x01e00000`, clock control `0x0a104008`. Recovered
register uses include start/framing `0x10`, opcode/address/dummy controls `0x14`,
address/length `0x18/0x1c`, busy `0x30`, FIFO count `0x34` and data `0x100`.
Commands include JEDEC ID `0x9f`, array read `0x03` and SFDP `0x5a`.

The original ID table at `0x264338` has 19 matched prefix/bound records. Entry
18 duplicates entry 16's prefix with another capacity; first-match behavior
selects the original 2 MiB bound. Table bounds are not physical capacity tests.

A stock `0x01000071` query accepts four bytes, band plus reserved, and returns
two signed path offsets plus reserved bytes. The reserved bytes are not extra
calibration values. Later live VRX work identified a version-4 factory record
at offset `0xa000` in the separate 2 MiB radio flash. The [P1 layout](p1-calibration-and-settings.md)
has different host-side storage.
