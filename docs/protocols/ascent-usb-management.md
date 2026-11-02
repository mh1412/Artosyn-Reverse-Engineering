# Ascent USB management and file-transfer protocol

[Protocols](README.md) · [Stock updater](../firmware/ascent-stock-updater.md)

The standard Ascent PC updater uses USB CDC ACM virtual serial. It does not
require SSH or the physical debug UART. The recovered Windows 2.0.40 and macOS
1.1.6 paths agree on framing; later 2.2.9 analysis retained the command family.
Receiver behavior was traced separately for 17.5.8 reference and live 17.5.3 /
18.21.10 binaries. Shared host framing does not establish identical target-side
flash behavior across models.

## Header

All fields in the 36-byte control header are little-endian:

| Offset | Size | Field / traced host value |
| --- | ---: | --- |
| 0 | 4 | Magic `0x4152544f`, wire bytes `OTRA` |
| 4 | 2 | Vendor `version` marker `0x0cdc` |
| 6 | 2 | Type: 1 request, 2 reply |
| 8 | 2 | Message ID; PC defaults to 0 |
| 10 | 2 | Unused marker `0xabcd` |
| 12 | 4 | Command |
| 16 | 2 | Format 2 |
| 18 | 2 | User ID 0 |
| 20 | 4 | Payload length |
| 24 | 4 | Sequence |
| 28 | 4 | Retry |
| 32 | 4 | CRC32 |

CRC32 covers the full header with bytes 32–35 zeroed, followed by payload.
It matches the standard `zlib.crc32` calculation. The field at offset 6 is a
packet type, correcting an early interpretation as a version.

An empty information request with message ID 0, sequence 1 and retry 0 has
CRC32 `0x4ac691be`:

```text
4f 54 52 41 dc 0c 01 00 00 00 cd ab 3c 00 00 00
02 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00
be 91 c6 4a
```

This serial header is distinct from the modern signed OTA header, compact
boot-image header and U-Boot HID envelope, despite overlapping naming/magic.

## Commands and transfer boundaries

| Command | Meaning | Typical payload / reply |
| --- | --- | --- |
| `0x3c` | Information | Empty request, 300-byte body |
| `0x03` | Reboot selection | `clean` padded to 32 bytes selects clean Linux startup |
| `0x72` | Prepare receiving session | Empty request, 68-byte status body |
| `0x73` | FileStart | 328-byte metadata, status body |
| `0x74` | FileData | Normally up to 1 MiB per chunk, 80-byte transfer status |
| `0x75` | FileEnd | Empty request, MD5 check and possible update trigger |
| `0x76` | Progress | Empty request, 72-byte progress/state/description body |

The host matches command/sequence acknowledgements, with observed two-second
retry and eight-second pending-ACK budgets. Progress polling is roughly every
500 ms. These are host implementation timings, not immutable protocol limits.

On 18.21.10, FileData validates CRC, writes each new remembered sequence once,
updates counters and flushes the file. Repeated remembered sequences are
acknowledged without another write. FileEnd closes the file and recomputes MD5;
its ACK is sent before creating the upgrade worker. An ACK therefore establishes
accepted receiving, not RSA acceptance, flash completion or successful boot.

The current traced FileEnd path can start upgrading for an exact `.img` extension
**or a size of at least 1 MiB**. Smaller non-`.img` transfers take a different
test/session-reset branch; this is not a general guarantee that FileEnd is
harmless. An internal diagnostic marker alters behavior but was not used by
these access tests. MD5 failure deletes staging and clears the session.

`0x72` resets receiving state; it does not itself run filesystem preparation or
the NAND writer. Device-specific [clean startup and flash execution](../firmware/ascent-stock-updater.md)
follow separate state transitions.

## FileStart metadata

| Offset | Size | Field |
| --- | ---: | --- |
| 0 | 64 | Lowercase ASCII MD5 hex, NUL-padded |
| 64 | 4 | File length; signed 32-bit field in the inspected host |
| 68 | 4 | `saveAsFile`, host sets 1 |
| 72 | 128 | Remote destination path |
| 200 | 128 | Local PC path, backslashes normalized to slashes |

Normal firmware staging requests `/tmp/pc/<basename>`. The receiver opens the
supplied path with `wb+` before later image authentication. Configuration upload
states can request factory destinations. This file-receiving interface is not
equivalent to an authenticated raw-flash operation, nor does it make a read-only
SquashFS file writable.

The inspected 17.5.8 receiver package is 49,207,658 bytes, requiring 47 chunks
at 1 MiB. Its MD5 is `4996760841a1bcd0332dea3b7e157542`, and complete-file SHA-256
is `f96497d662f4a6223482d55bdb903687a2e660e4759eecb152a91fa29f41ceb5`.
Transport CRC, received-file MD5 and signed-body SHA/RSA serve different layers.

## Information reply

The examined 300-byte reply parser identifies:

| Offset | Size | Meaning |
| --- | ---: | --- |
| 0 | 4 | Maximum transfer bytes |
| 4 | 32 | Application version text |
| 36 | 64 | Device name |
| 100 | 4 | Signed temperature value, interpreted as Celsius in the tested getter |
| 104 | 64 | Firmware text |
| 168 | 32 | Serial text |
| 200 | 32 | Hardware text |
| 232 | 4 | Signed status |
| 236 | 64 | Detail text |

Text fields are NUL-terminated within their bounds. Identity and valid CRC
establish communication with that service; they do not establish radio or video
health.

## Other transports and contention

Stock Linux normally presents RNDIS plus CDC ACM as `1d75:0101`. Its optional
mass-storage branch creates a 45 MiB RAM FAT staging disk, not a whole-chip flash
export. U-Boot's `1d6b:0100` HID path is a separate
[RAM-loading protocol](../firmware/gnd485-ram-boot.md).

The stock USB script appends an `askfirst` shell for `ttyGS0` and signals init.
When an account overlay makes `/etc` writable, that append can succeed where
the original read-only root rejected it. A console shell and updater can then
compete for the same CDC stream, producing mixed protocol/console bytes.
USB Ethernet remains a separate interface.

fpvOS reuses OTRA transport but confines receiving to private RAM destinations
and accepts its own `.fwu` format. It is not a drop-in vendor signed-image engine;
see [fpvOS update contracts](../firmware/gnd485-fpvos-update.md).
