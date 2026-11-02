# Gnd485 fpvOS installation and recovery contracts

[Firmware](README.md) · [NAND/GPT](gnd485-nand-and-gpt.md) · [USB RAM recovery](gnd485-ram-boot.md)

The main Gnd485 source profile boots Linux 6.18.36 from NAND and runs its full
userspace from RAM. Its kernel contains an independent recovery initramfs that
checks the installed bundle, kernel and DTB before extracting userspace to tmpfs.
Normal subsequent boots need neither a host nor an SD card. Runtime system-file
edits are ephemeral; SD supplies separate persistent storage.

This design differs from the experimental stock-kernel root-switching profile
described below and from the [P1 updater](p1-boot-and-recovery.md).

## Writable ranges and bank selection

The custom updater writes shared kernel and DTB plus the root bank not selected
as stock by validated vendor GPT. It never updates GPT, environment, bootloader,
factory, the active stock root or vendor-data ranges. Its master/protected
descriptor is read-only.

The first implementation assumed stock userapp0. Current kernel/updater code
shares a read-only GPT decoder and maps either physical bank to the `fpvos`
role. Installing on a stock-userapp1 system additionally requires a bank-aware
device tree; old fixed-map bundles are rejected before writes.

The inactive 21 MiB bank begins with two 128 KiB control eraseblocks. Bundle
data starts at bank-relative `0x40000`, leaving 21,757,952 bytes. Existing bad
blocks are rejected in an operation's affected ranges; the writer does not skip
or remap them. Boot checks only used records/images, so unrelated unused tail
blocks need not prevent boot.

Recovery readiness checks identity, geometry, layout and storage ownership
before the operation is known. A damaged fpvOS control/header area does not
block vendor restoration, which uses only kernel/DTB. Install still checks all
erase ranges before touching any NAND. This distinction corrected an earlier
preflight that rejected unrelated bad blocks.

## Bundle format

The fpvOS `.fwu` package uses `FPVOSUP1`, not the signed vendor OTA layout.
Its 512-byte header contains:

| Offset | Encoding / meaning |
| --- | --- |
| 0 | Eight-byte `FPVOSUP1` magic |
| 8 | LE32 format version 1 |
| 12 | LE32 board ID, 485 for Gnd485 |
| 16 | LE32 complete bundle length |
| 20 | LE32 component count |
| 24 | 64-byte NUL-terminated descriptive field |
| 88 | LE32 kind: 0 install, 1 vendor restore |
| 92 | Four reserved zero bytes |
| 96 | Up to three 104-byte component entries |
| 408 | 100 reserved zero bytes |
| 508 | CRC32 of the header with this field zeroed |

Each component entry has a 32-byte zero-padded fixed name, LE32 length, 64
lowercase ASCII SHA-256 characters and four zero bytes. Components follow the
header consecutively in entry order; unused entries must be zero. The complete
length must equal 512 plus component lengths.

| Install component | Maximum bytes |
| --- | ---: |
| `kernel.img` | 6,291,456 |
| `ascent-gnd485.dtb` | 393,216 |
| `rootfs.cpio.uboot` | 16,777,216 |

The normal complete bundle limit is the available 21,757,952-byte bank space.
Restore has exactly two components, each the complete original partition size,
for a 6,685,184-byte package. Component hashes and board/image checks provide
integrity and compatibility checks; there is no signature or version-order
policy in this format.

The kernel has a compact 64-byte OTRA wrapper and legacy uImage. Source load/entry
is `0x20200000`; vendor restoration expects `0x200a0000`. Header/data CRCs,
wrapper length/checksum and bounded padding are checked. The root component is
a U-Boot ramdisk containing gzip data. DT validation walks the structure and
requires the correct root compatible: source `caddx,ascent-gnd485` or vendor
`artosyn,proxima-9311`. This is separate from merely finding a string anywhere
inside arbitrary input.

## Installation transaction

The receiver bounds uploads before staging and restricts destinations to its
own private RAM area. It checks CRC/MD5 at the transport layer, then the NAND
transaction creates its own private read-only package snapshot. Verification,
programming and readback use that snapshot. Merely opening the original staged
file once would not prevent another writer from changing it in place.

### Native serial receiver

The source receiver uses the 36-byte OTRA serial header and limits a packet body
to 1 MiB. Its handling is distinct from the
[vendor receiver](../protocols/ascent-usb-management.md):

| Command | Native receiver behavior |
| --- | --- |
| Information `0x3c` | Reports updater identity and, when available, the installed bundle SHA-256 |
| Prepare `0x72` | Requires clean mode; resets transfer state and removes the previous status record |
| Metadata `0x73` | Requires 328 bytes, nonzero board-bounded size, file flag 1, lowercase MD5 plus zero padding, and an accepted install/verify destination token |
| Data `0x74` | Appends within the declared size; acknowledges received and total byte counts |
| End `0x75` | Requires exact received size, fsync/close and matching file MD5; verifies synchronously or starts the separate installer |
| Status `0x76` | Reports transfer/verification/installation state independently of the FileEnd ACK |

Destination strings select fixed operations; they do not choose the actual
staging path. The receiver creates `package.fwu` in its private, owned directory
using exclusive creation and no symlink following. Startup also takes a
nonblocking lock for that staging directory.

Only the last data sequence is remembered. A repeat with the same sequence,
payload CRC32 and length is acknowledged without another append; changed
identity is refused. Later sequences are accepted by a signed 32-bit forward
difference, so gaps are permitted rather than requiring an increment of one.
Older sequences are refused. This is a checksum-based retry identity, not a
stored-byte comparison or a cross-reconnect transfer-resume protocol. Whole-file
MD5 remains a separate final check.

FileEnd's “transfer verified” ACK does not mean installation succeeded. Verify-only
finishes at state 2/90%; install enters state 3. State 4/100% requires both the
worker's successful exit and its completed status report. While a worker runs,
information/status requests remain available and mutations are refused. State
−2 represents failed restoration and blocks normal reboot/update commands.

The I/O helper uses a 30-second poll for each remaining read/write portion,
resetting the wait after progress. It is not a whole-transfer deadline. Helper
execution and waiting for an active installer have no overall deadline in this
receiver. Disconnect waits for the installer; it does not cancel programming.
Completed/error status can survive a receiver restart in the retained RAM
directory, but that is not durable recovery across power loss.

### NAND programming

After locking the operation, the backend checks the exact board/runtime,
partition geometry, GPT and UBI ownership. It rejects unknown attachments or
UBI attached to a writable target. A stock-kernel path can retain known
read-only stock/factory/data attachments; it does not treat every attachment
as safe.

The install order is:

1. Validate the private package and all affected erase ranges; check RAM space.
2. Back up the full shared kernel/DTB and inactive bundle-data region in RAM.
3. Erase/write/readback the root bundle, writing its first eraseblock last.
4. Erase/write/readback DTB, then kernel.
5. Write a normal-mode control record and report installed/readback-verified.

Page writes pad with `0xff` and skip entirely erased pages. Readback compares
bytes and checks that uncorrectable ECC counters have not increased. A reported
write failure triggers restoration of the affected saved ranges. Failed
restoration preserves RAM evidence and blocks normal service updates/reboots.
Disconnecting the USB client does not cancel an active flash transaction.

**Shared kernel/DTB programming is not power-loss atomic.** RAM backups cannot
survive a power cut. An invalid root leaves independent recovery available if
kernel/DTB still boot; damaged shared boot images can require host USB RAM
loading. No live power interruption during programming was part of the recorded
acceptance tests.

## Clean-mode records and boot checks

Each control slot uses one 2 KiB page at the beginning of its eraseblock:

| Offset | Field |
| --- | --- |
| 0 | `FPVOCTL1` |
| 8 | LE64 sequence, nonzero |
| 16 | LE32 mode: 0 normal, 1 clean |
| 20 | CRC32 over the page with this field zeroed |
| Remaining bytes | Initialized to `0xff` |

The newest valid sequence wins. Equal valid sequences are rejected as a conflict,
and sequence overflow is refused. A new record erases/writes the opposite slot
and verifies its page. I/O errors and conflicting records must not be treated as
an uninitialized area that permits erase.

On clean boot, recovery attempts to consume the flag back to normal and stays
in recovery. On normal boot it validates the stored bundle, extracts components
and compares actual flash kernel/DTB to the package before starting full userspace.
This prevents an old root bundle from silently being accepted after restoring
different vendor boot images.

The host's ordinary update flow requests clean mode, reconnects after recovery
reboot, uploads/installs, reboots and verifies the reported bundle hash. A
RAM-loaded development system instead quiesces clients/unmounts storage without
needing the same NAND clean reboot. A verify-only response at 90%, state 2 is
not installation; state 4/100% reports completed readback verification.

## Vendor restoration

Restore packages contain complete original kernel and DTB partitions, including
non-erased trailing bytes. A trimmed embedded image would not restore partition
identity. The documented original partitions were:

| Partition | SHA-256 |
| --- | --- |
| Kernel, 6 MiB | `281da6c6143d81a416e73c9063767a3da8dd0c05813d3e95b666798ac730a7b2` |
| DTB, 384 KiB | `213d097d407e8473d928338dd4c8168d5657561424043b2502748c1287a1b257` |

The matching full restoration bundle SHA-256 is
`2f2f9c1917c39304cbd20a4b0b6250a7037fb2351fc22c1bf9a24abd4722cfd3`.
The current restore transaction neither reads nor programs either root bank's
data/control area. Hardware checks from stock preparation and native recovery
confirmed preservation of both banks and changes only to kernel/DTB.

A restored vendor GUI/USB identity does not establish all networking/RF/video
functions. Early restoration boots lacked the expected ECM/SSH connection;
later regression confirmed stock USB identity and GUI startup. Stock software
can also change factory/data after boot, so whole-device byte identity is a
different claim.

## Stock clean-hook bootstrap

The unified static updater embeds its preparation tools. A small separate shell
launcher is still needed because stock's `web:path` hook sources a script rather
than executes a binary. The staged updater/launcher are verified, then a consumed
one-shot selector requests this path on a normal reboot. The transfer deliberately
omits vendor FileEnd; the custom receiver, not the vendor flash engine, handles
the later `.fwu` install.

Preparation validates the matched Linux 4.9.38/Gnd485 runtime, moves execution
into private tmpfs, removes staged copies and checks mounts using PID 1's view
outside its chroot. Missing host mount information fails readiness. If readiness
fails, USB information/status remains available for a clean retry, while uploads
are refused. Initial preparation itself does not install firmware or export host
backups.

## Hardware validation and historical alternative

October 29 tests of source `5644edc` and bundle
`1a7df87195cb7f220ae68e9afc54c8094a0ae9b77f3b32ae680d49a6932b6efe`
covered stock clean-hook preparation, oversized/corrupt rejection, NAND install,
native recovery self-update, exact cold boot and vendor restoration. Both roots
were preserved by restore and ECC failure counters remained zero. That run used
stock bank 0, an incremental build and a Python protocol client; broader earlier
development reported both bank selections and a browser round trip. Physical
bad-block, staged-file mutation and interrupted-write cases remained simulated.
Later [combined platform regression](../experiments/ar9311-platform-validation.md#later-installed-platform-regression)
adds full platform/decode/storage checks.

The separate **stock-kernel profile** retained vendor Linux 4.9.38/kernel/DTB,
stored fpvOS as UBI/gzip SquashFS in userapp1, copied userspace to RAM and detached
UBI before updating. It selected stock first, verified the new fpvOS bank, then
committed a newer GPT record with the header last. This gave update fallback,
not boot-health rollback or retention of the previous fpvOS release.

That profile passed root-only USB installation, stock/fpvOS selection and cold
boot, plus 29 host tests including twelve simulated power-cut points. It did not
include the custom 6.18 media/HDMI/radio modules. The main project later returned
to the full 6.18 platform. Initial failures involving MTD `fsync`, wrong backup
GPT addresses, stale RAM helper and a concurrent status lock were corrected;
they are not successful update trials. The two designs must retain separate
GPT, image and recovery semantics. The stock-kernel build profile and dedicated
backend were subsequently removed on October 28; the historical tests remain
research evidence, not a currently maintained alternative target. Vendor
kernel/DTB restoration remains part of the full-platform updater.

## Recorded October 31 prerelease

The `2026.10.31-ascent-vrx.1` prerelease was built from source commit
`8cb13d6eed4e5b913625f79f675dfd957825cb18`. Its install bundle SHA-256 is
`fc76aa1fe7c702176063c15b7eee87f88b9df2f2f0eb0fd4aa45513bda108ff4`.
Recorded component lengths were 5,228,544 bytes for the kernel image, 8,700 for
the DTB and 5,558,938 for the root image, after the 512-byte bundle header.

The release audit checked 75 updater tests, ten extraction tests, C verification,
browser bundle/bootstrap parsing, 502 rootfs entries and 13 uploaded asset hashes.
The decoder/radio firmware matched previously tested files, and the static
updater matched the rootfs copy. At publication, this exact build had not yet
been hardware retested and the website bootstrap pin was unchanged.

A later October 31 deployment and
[hosted-browser hardware run](../experiments/gnd485-browser-update.md)
installed this exact bundle, completed self-update, verified its hash after a
power cycle and restored stock 18.21.10 on Gnd485 V1.2-1.0. The run used stock
bank 0 and recorded zero platform-check failures, with explicit media/storage
test limits. Its bootstrap manifest lists stock versions 17.5.3, 17.5.8 and
18.21.10; that declaration does not add three physical validation runs.
