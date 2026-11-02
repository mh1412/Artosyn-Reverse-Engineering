# Ascent Lite ASW update engine and bank selection

[Firmware](README.md) · [Receiver updater](ascent-stock-updater.md) ·
[Sky482 NAND](sky482-nand-layout.md)

The October 7, 2026 offline study traced the actual H Sky 17.5.8 application
updater, `ar_fpvhs_upgrade`. It shares the Ascent serial upload frontend with
the receiver but has different CPU/libc requirements, image validation and flash
selection. No air-unit update was executed in this study. Later 18.21.10
[direct-flash recovery](../experiments/ascent-lite-flash-repair.md) is a separate
experiment, not validation of this complete updater path.

## Reference identity

| Artifact | Bytes | SHA256 |
| --- | ---: | --- |
| H Sky 17.5.8 image | 11,509,044 | `b617532697eccac4bb7b2ddbbcbf2bc454ba04561ca370a6cd5ef860a3554b51` |
| `ar_fpvhs_upgrade` | 341,056 | `ad7f87e6eeca7b75fec5eb7268e880e13f71bd2f0d0def306a1e66653c1558ab` |
| Root UBI component | 4,063,232 | `98337f11716208485c9428c577d29be1748230b3158178cfe33e473827b579d8` |
| Application UBI component | 6,946,816 | `d1e3d8ea523b78f9cba7aa95244734b8fc67766804734f0f47cf5e5a0ffcb524` |

The extracted root identifies `APP_VERSION=17.5.8`, `BOARD_TYPE=Sky482` and
build `202604201925`. The updater is little-endian ELF32 ARM EABI5/Thumb,
using `/lib/ld-musl-arm.so.1`. SKY-related functions in the receiver's different
updater were an investigative lead, not proof that Lite runs that executable.

## Direct USB frontend and clean runtime

The inspected PC-tool flow uploads to the directly attached VTX. The name
`RemoteUpgrade` for command `0x72` does not establish an RF update through a VRX.
The [36-byte OTRA wire framing](../protocols/ascent-usb-management.md) carries:

| Command | Role |
| --- | --- |
| `0x3c` | Device identification |
| `0x03` | `clean` padded to 32 bytes; clean-mode reboot |
| `0x72` | Prepare/reset receiving session |
| `0x73` | 328-byte metadata with MD5, length, save-as-file flag and paths |
| `0x74` | Acknowledged chunks, normally 1 MiB |
| `0x75` | FileEnd, transfer validation and updater start |
| `0x76` | Progress query |

The normal transfer destination is `/tmp/pc/` plus the selected image basename.
PC Tool 2.2.9's Lite/Lite+ default reconnect delay is 8,000 ms; that setting is
not a measured device boot time. Its older-Sky-Z8 guard around version 16.4.4
and other multi-image dispatch paths prevent treating every Sky product as Lite.

Reference init runs `ar_bsel check_cmdline`, mounts filesystems and checks
`/factory/sirius-clean-system-flag`. Normal board-482 startup launches
`ar_fpvhs_upgrade` with board/hardware arguments. Normal serial configuration
exposes CDC ACM, `1d76:0101`, serial `CX20`, and the updater opens `/dev/ttyGS0`.
This differs from the standard receiver's combined serial/network gadget.

Clean startup copies the board helper, serial script, updater and reboot utility
to RAM, unmounts `/usrdata` and `/factory`, creates `/tmp/pc`, and runs
`/tmp/ar_fpvhs_upgrade -c 0 -m 1`. FileEnd syncs/closes the received file, compares
its transfer MD5 and dispatches the upgrade thread. Board type 3 selects the
Sky48x path. RAM execution removes mounted application-file dependencies but
does not make the eventual flash operation atomic.

## ASW format in this package

The image contains a **128-byte ASW header** and no `OTRA` occurrence. OTRA
serial framing therefore must not be confused with an OTRA firmware envelope.

| Header offset | Encoding | Observed field |
| --- | --- | --- |
| 0 | 4 bytes | `ASW` followed by zero |
| 4 | LE32 | Board type, 3 |
| 8, 12, 16 | Three LE32 values | Version components, 17 / 5 / 8 |
| 20 | Five 12-byte descriptors | Component metadata |
| 112 | LE32 | Payload CRC32, `d45a474e` |
| 116 | LE32 | Declared file size, 11,509,044 |
| 120 | LE64 | Decimal build timestamp, 202604201925 |

Each descriptor contains four one-byte fields (type, upgrade-enable, dual and
reserved), a legacy LE32 field at +4 and a LE32 source offset at +8. The writer
uses descriptor **index** for destination selection. Source extent ends at the
next descriptor's offset, or EOF for the fifth. The legacy +4 values are not
the actual component payload sizes and must not be used as carving lengths.

| Index | Raw type | Enabled | Source offset | Payload bytes | Component |
| ---: | ---: | ---: | ---: | ---: | --- |
| 0 | 0 | 1 | 128 | 236,544 | Boot |
| 1 | 1 | 1 | 236,672 | 262,144 | Environment |
| 2 | 2 | 0 | 498,816 | 180 | Disabled kernel entry |
| 3 | 4 | 1 | 498,996 (`0x79d34`) | 4,063,232 | Root UBI |
| 4 | 5 | 1 | 4,562,228 (`0x459d34`) | 6,946,816 | Application UBI |

All five dual fields were 1. The actual write mapping below still includes
shared boot/environment destinations. The disabled 180-byte kernel entry does
not supply a complete kernel for reconstruction. Root and application are
different volumes; the latter contains the updater, startup scripts and radio
files.

The offline audit checked monotonic bounded source ranges, declared size and
standard CRC32 over `[128, EOF)`. That audit is distinct from the device's own
validation behavior.

## Validation and write call chain

```text
FileEnd transfer MD5
  → AR_FPV_UPGRADE_UpgradeCreateThread
  → AR_FPV_SKY48x_UpgradeProcessThread
  → image search, board/header/filename version checks
  → AR_FPV_UPGRADE_SKY48x_ImgUpgradeProcess
  → AR_FPV_UPGRADE_SKY48x_ImgUnpack
  → unpack_images_mmap
  → AR_FPV_UPGRADE_SKY48x_ImgWrite
  → AR_FPV_UPGRADE_SKY48x_ImgWriteFlash
  → nand_flash_write_normal
```

For a nonzero header CRC, as in the official package, the version-check path
calls the payload CRC verifier, which reads in 4 KiB chunks. The complete traced
user-space path contains transfer MD5, payload CRC32 and board/version checks;
no SHA-256/RSA verification or external public-key-loading call was observed.
The receiver's public-key-file path was absent from these extracted trees.

MD5 and CRC32 establish integrity, not a firmware signature. This result does
not settle BootROM, bootloader, radio authentication, newer firmware or other
air-unit products. No modified package was submitted to establish acceptance
or successful boot.

`unpack_images_mmap` at `0x146ec` writes components while iterating. It is not
a passive extractor. The saved disassembly also shows two limits on treating
its progress report as complete validation:

- Bounds checks occur during iteration, after earlier components may have been
  written. At `0x14898`–`0x148b4`, the out-of-bounds branch logs and advances to
  the next descriptor, rather than immediately returning an error. This is a
  static control-flow observation, not a malformed-image hardware test.
- After the application write, `system("ar_bsel set 0|1")` is followed by logging
  without checking that call's return value in the observed function. The
  command invocation alone does not verify durable bank selection.

Negative component-write results follow an error-return path. The NAND helper
queries geometry/capacity, checks bad blocks, erases/programs through MTD and
has mark-bad/error handling. Those branches are not a power-interruption test.

## Targets and commit boundary

| Descriptor | Shared destination | If `/run/bootsel0` exists | Otherwise |
| --- | --- | --- | --- |
| Boot | `/dev/mtd0` | Same shared target | Same shared target |
| Environment | `/dev/mtd1` | Same shared target | Same shared target |
| Kernel, when enabled | — | `/dev/mtd4`, kernel1 | `/dev/mtd3`, kernel0 |
| Root | — | `/dev/mtd7`, rootfs1 | `/dev/mtd6`, rootfs0 |
| Application | — | `/dev/mtd9`, usrdata1 | `/dev/mtd8`, usrdata0 |
| Selection command after application | — | `ar_bsel set 1` | `ar_bsel set 0` |

After the component loop, the success path removes the staged package and syncs;
the surrounding updater reports completion, consumes its upgrade-mode state and
reboots. Shared boot/environment writes remain outside the paired bank scheme.
This is not a whole-system atomicity claim, and the receiver's GPT/partition
numbering must not be substituted.

The next startup loads AR8030 image/configuration files from the application
filesystem. Updating those stored files is distinct from a demonstrated permanent
write to the radio chip itself.

## Other product workflows

The archived Lite guide describes direct PC-tool USB updating with no external
battery needed for that model. The Lite+ guide also describes wired PC updating
but does not independently state the same no-battery condition in its upgrade
section. The archived Avatar V2 guide instead describes copying an
`Avatar_Sky_X.X.X.img` file to the VTX USB disk and holding LINK for eight seconds.
Those are model-specific documented frontends, not live tests in this study;
Avatar's target updater/trust chain was not extracted.

The H Sky binary also contains GT/Sky4862 and host/slave multi-image branches,
including internal connections using `192.168.4.10`/`.20` and ports 8888/8889.
Their presence does not prove firmware delivery over the stock RF link or
justify extending the board-3 validation findings to every branch.
