# P1 stock boot and NAND findings

[Firmware](README.md) · [P1 Air](../devices/betafpv-p1-air.md)

Scope: the investigated P1_SKY 1.0.44 layout. These observations are not a
model-independent flashing recipe or a complete stock partition inventory.

## Recorded NAND regions

| Region | Offset | Size |
| --- | --- | --- |
| GPT selection records | `0x100000` | `0x80000` total |
| Factory | `0x200000` | `0x600000` |
| Kernel 0 | `0xa40000` | `0x600000` |
| Kernel 1 | `0x1040000` | `0x600000` |
| DTB 0 | `0x1640000` | `0x60000` |
| User application 0 | `0x1700000` | `0x2d00000` |
| User data | `0x7100000` | `0x600000` |

The installed stock Linux lacked both kexec entry points. The stored device tree
and live memory range differ; see the [device study](../devices/betafpv-p1-air.md).

## Bootloader observations

The tested U-Boot path rejected legacy LZ4 and accepted a standard LZ4 frame
with independent blocks in the expected OTRA/uImage form. RAM load and entry
for that tested image path were `0x20200000`; they are not universal image
addresses.

The volatile SPL USB-request register is `0x0a106138`. Tested value 3 enters
the U-Boot USB downloader; value 2 clears request bit 0. Zero is unsuitable
for that reboot-mode implementation because it ignores zero.

A boot-state comparison found that U-Boot left a 150 MHz QSPI clock source;
using it without the required controller setup produced shifted NAND identity
reads. A 25 MHz source and reset sample delay restored detection in the probe.
This identifies an inherited-state dependency, not proof that stock Linux uses
that reduced clock or a general recovery procedure.

Independent recovery from damaged SPL/U-Boot and complete stock update failure
behavior remain unqualified. Firmware images, raw NAND/OOB backups and selected
partition readbacks are different artifacts.
