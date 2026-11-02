# OTRA containers and extraction-tool boundaries

[Firmware](README.md) · [Documentation index](../../README.md)

The retained ar-firmware-tools implementation describes an OTRA package family
and supplies an offline extractor. Its README calls the subject “ARTO v4,” while
the schema and dispatch code expect the four literal bytes `OTRA`. This is a
container signature, distinct from the similarly named
[Ascent USB management framing](../protocols/ascent-usb-management.md).

This page records the reviewed parser and helper behavior. It does not establish
that every vendor image uses this layout, or that an extracted/repacked image
will pass a device's updater authentication.

## Normal container layout

The schema uses little-endian integers. The fixed header is 64 bytes, followed
by a variable-length hash and signature. Its comments primarily scope the
later fields to version 4, although code includes branches for older versions.

| Offset | Size | Parsed field |
| --- | --- | --- |
| `0x00` | 4 | `OTRA` magic |
| `0x04` | 2 | Hash length in bytes |
| `0x06` | 2 | Signature length in bits |
| `0x08` | 8 | Image size |
| `0x10` | 4 | ROM payload size |
| `0x14` | 4 | Loader payload size |
| `0x18` | 4 | Environment payload size |
| `0x1c` | 2 | Version-field size |
| `0x1e` | 2 | Partition count, table 0 |
| `0x20` | 2 | Compression flag |
| `0x22` | 2 | Segment count, table 0 |
| `0x24` | 1 | Partition count, table 1 |
| `0x25` | 1 | Segment count, table 1 |
| `0x26` | 1 | Flash type |
| `0x27` | 1 | Header version |
| `0x28` | 2 | Actual-signature-size field |
| `0x2a` | 4 | Object version |
| `0x2e` | 4 | Dependency version |
| `0x32` | 14 | Reserved bytes |
| `0x40` | Variable | Hash bytes, then signature bytes using declared bit length / 8 |

For header versions greater than 3, the parser next skips 224 bytes and reads
a 32-byte extension: 16 bytes for a NUL-terminated SDK version and 16 bytes of
partition flags. ROM, loader and environment payloads follow. Versions greater
than 1 then have partition tables 0 and 1, followed by segment tables 0 and 1.

Each 52-byte partition entry contains a 32-byte name, 64-bit flash offset,
64-bit length and 32-bit upgrade flag. Each 32-byte segment entry contains
four 64-bit values: image offset, flash offset, compressed size and decompressed
size. Segment payloads are read at their absolute image offsets. The version
size field is parsed, but the schema's separate version-body accessor remains
commented out.

## What extraction reconstructs

The normal-image command writes ROM, loader and environment blobs when present.
It assigns a segment to a partition when the segment's **starting** flash offset
lies inside that partition, then concatenates matching segments in table order.
With the package compression flag and a nonzero decompressed size, it applies
LZO decompression to each segment.

This is not a sparse flash-image reconstruction: it does not place segments at
their relative flash offsets or fill intervening gaps, and the assignment test
does not establish that the entire segment fits the partition. Those details
matter when interpreting the resulting `*.img` files as evidence about storage.

Normal-image `info` displays the stored hash and signature; the reviewed command
does not cryptographically verify them. Merely extracting these fields is not
an authentication result. Device-side checks belong to the separately studied
[stock updater](ascent-stock-updater.md), with its own model and version scope.

## SPL and Sirius dispatch

The command also recognizes separate SPL and Sirius layouts using patterns of
zero and nonzero header words after the common magic. These are heuristic
dispatch rules, not a universal OTRA version registry. SPL extraction emits
SPL, `troot` and signature components. Sirius extraction emits its image
payload as `rom.img`.

Sirius `info` computes an additive byte checksum and prints a mismatch rather
than refusing extraction. The `pack sirius` command writes a 64-byte envelope
with magic, load address, payload length and that checksum. It does not create
a vendor signature. Successful packing therefore demonstrates file generation,
not a flashable, authenticated replacement firmware.

## Deep extraction and provenance loss

The Avatar helper first extracts the top-level image, then examines immediate
child `*.img` files for OTRA magic. When a child produces `rom.img`, that payload
replaces the child file. Despite the helper's recursion-oriented name, this is
one child pass rather than an arbitrary-depth traversal.

Kernel images are passed through an LZ4/initramfs helper; user-application
filesystems are extracted into `filesystem/local`, and the UI application into
`filesystem/gui`. Filesystem type detection selects JFFS2 or ext4 tooling;
DTBs are converted into DTS text. Multiple user-application images share the
same output directory, so the combined tree can lose bank/partition provenance
and overwrite equal filenames. Preserve individual components when comparing
versions or banks.

The LZ4 helper searches for legacy magic `02 21 4c 18`, first near the beginning
of the kernel image and then inside the decompressed kernel. Both decompression
commands and the final CPIO extraction explicitly tolerate failure. Partial
output or a successful shell exit does not establish a complete filesystem.
The outer helper also deletes an existing destination directory before moving
results into it; it is not an incremental, evidence-preserving merge.

These conclusions come from source inspection. No extraction helper or device
updater was executed for this review. See
[firmware components and update paths](architecture-and-updates.md) for the
separate ASW, raw NAND and board-specific installation boundaries.
