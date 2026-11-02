# Gnd485 U-Boot access and USB RAM loading

[Firmware](README.md) · [NAND and environment](gnd485-nand-and-gpt.md)

The standard AR9311 Ascent VRX has a verified U-Boot USB RAM-transfer and boot
path. Early October reports described only compiled commands; later October 25–26
work established command access, USB upload/readback and kernel boot. This is
separate from the stock Linux CDC firmware updater and from AR8030 radio loading.

## Console and authorization milestones

UART0 is approximately 1.8 V, 1,152,000 baud, 8N1. An earlier adapter rounded
the requested baud to 1,000,000 and did not establish interactive output.
Autoboot interruption and command authorization are separate: the studied
U-Boot dispatch hashes the first command token with MD5 and compares a stored
digest before enabling commands.

The recorded development path first persisted a five-second boot delay, then
changed the selected U-Boot's password-authentication data. The October 25 result reports programming only the first
128 KiB of U-Boot0, full readback verification, zero ECC failures and successful
`version`/environment commands after boot. U-Boot1 and the checked SPL, GPT,
BL31 and environment regions were unchanged. Automatic fallback to U-Boot1 and
an end-to-end restoration of original U-Boot were not established by that test.

The resulting U-Boot partition SHA-256 was
`29c2e4668bade9ecf5807982828353d173d770c9297f848c59d93b3832199e2d`.
Later RAM-boot validation used that established command access. These findings
do not imply that an unmodified retail unit already exposes an unlocked prompt.

## HID RAM transport

Entering `artosyn_hid_upgrade 0` on the unlocked console exposes:

| Property | Observed value |
| --- | --- |
| VID:PID | `1d6b:0100` |
| Manufacturer / product | `U-Boot` / `USB download gadget` |
| Interface | 0, HID class |
| OUT / IN endpoint | `0x02` / `0x81`, 1,024-byte packets |
| RAM upload / read operation | 4 / 5 |
| Return to console | UART Ctrl-C |

The bounded host implementation uses at most 960 body bytes. Its frame has a
36-byte outer header followed by a 12-byte little-endian inner header:

| Offset | Field |
| --- | --- |
| 0–7 | Magic bytes `ff 55 ff aa 41 52 54 4f` |
| 8–15 | Zero in the host implementation |
| 16 | LE32 total frame length |
| 20 / 24 | LE32 fragmentation fields 1 / 0 |
| 28 | Byte 1; remaining outer bytes zero |
| 36 | LE16 operation |
| 38 | LE16 inner-header size 12 |
| 40 | LE32 body length |
| 44 | LE32 zero |
| 48 onward | Body |

Upload setup is ASCII `uftr`, LE32 total length and LE64 destination address.
Subsequent operation-4 bodies carry data. The vendor handler interprets any
16-byte body as a possible setup command, so the uploader splits a final
16-byte data tail into two eight-byte transfers.

Read requests use operation 5 with `ufgt`, LE32 requested length and LE64
address. The response's inner operation is 2, with payload length matching the
request and the same checked fragmentation values. The bounded tool implements
only RAM operations 4/5; it does not use the separate flash-update operation.

## RAM windows and boot sequence

The examined host loader restricts complete transfers to these half-open ranges:

| Object | Allowed RAM window |
| --- | --- |
| Kernel staging | `0x24400000–0x24c00000` |
| DTB | `0x22080000–0x220e0000` |
| Initramfs | `0x26000000–0x26800000` |

The first 32-byte marker test verified USB upload, USB readback and independent
UART CRC `497bdb18`, then restored the original bytes. The next stage uploaded
a full 6 MiB installed vendor kernel partition and 384 KiB DTB into cleared RAM,
with byte equality and UART CRC checks. Transfer plus readback took about
6.65 seconds for the kernel and 0.433 seconds for the DTB in that setup.

`bootm 0x24400000 - 0x22080000` then booted vendor Linux 4.9.38 and its NAND
userspace, with UART login, USB networking/SSH and backend startup verified.
This first boot used the installed vendor kernel, not a source-built kernel or
self-contained RAM OS. The later fpvOS source kernel and external initramfs
completed those separate milestones.

For the source profile, temporary U-Boot settings include `mem=256M`,
`rdinit=/init`, `fdt_high=0xffffffffffffffff`,
`initrd_high=0xffffffffffffffff`, kernel staging at `0x24400000` and an 8 MiB
compression workspace limit. After verified transfers, the recorded command is:

```text
bootm 0x24400000 0x26000000 0x22080000
```

The explicit memory limit keeps Linux out of the upper codec pool despite
U-Boot memory fixups; keeping the initramfs in place avoids unsuitable relocation.
These are temporary boot settings. The custom kernel's load/entry is
`0x20200000`, while the tested vendor kernel loads at `0x200a0000`.

## SD boot and other compiled paths

The earlier command-table audit found 83 entries, including filesystem loaders,
`bootm`, `booti`, `bootelf` and serial loaders, but no `mmc` command. Later live
capability inspection also found no MMC driver. Generic filesystem command
presence and an MMC-shaped saved boot command did not establish access to the
card slot. Linux SD persistence and manufacturer SD-to-NAND updating are
separate capabilities.

U-Boot also contains USB-storage, RAM-image and HID flash-update routes. Their
existence does not make the validated RAM transport a flash recipe. A static
trace of its OTA core did not find the same SHA/RSA calls as the Linux updater,
but that is not a hardware-proven modified-package acceptance or a statement
about every boot trust boundary.

USB re-enumeration and host interface addressing caused failed attempts in the
recorded work. Successful boot requires verified bytes and the appropriate
effective arguments, not merely an accepted upload. For installed fpvOS,
USB RAM boot remains the independent recovery route if shared kernel/DTB
programming is interrupted.
