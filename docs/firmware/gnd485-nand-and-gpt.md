# Gnd485 NAND layout, GPT and environment

[Firmware](README.md) · [Ascent VRX](../devices/ascent-vrx.md)

The studied standard Ascent Gnd485 uses 128 MiB SPI NAND with 128 KiB eraseblocks,
2 KiB pages and 128-byte OOB. Stock Linux exposes a whole-chip master as well as
partitions. Its MTD numbers differ from the source fpvOS map, which hides the
master and gives protected regions read-only roles. Partition names and physical
ranges must be checked before interpreting a device number.

## Physical layout

| Region | Offset | Size |
| --- | --- | --- |
| SPL copies 0–3 | `0x000000`, `0x040000`, `0x080000`, `0x0c0000` | `0x40000` each |
| GPT records 0/1 | `0x100000`, `0x140000` | `0x40000` each |
| Vendor | `0x180000` | `0x80000` |
| Factory | `0x200000` | `0x600000` |
| BL31 | `0x800000` | `0x80000` |
| Environment 0/1 | `0x880000`, `0x8e0000` | `0x60000` each |
| U-Boot 0/1 | `0x940000`, `0xa40000` | `0x100000` each |
| Shared kernel | `0xb40000` | `0x600000` |
| Shared DTB | `0x1140000` | `0x60000` |
| User application 0/1 | `0x11a0000`, `0x26a0000` | `0x1500000` each |
| User data 0/1 | `0x3ba0000`, `0x51a0000` | `0x1600000` each |
| FPV data | `0x67a0000` | `0x1600000` |

The last named partition ends at `0x7da0000`; this is not a claim that every
remaining byte is unused. fpvOS groups the range from `0x3ba0000` through flash
end into its protected vendor-data region.

Stock updates combine shared partitions with adjacent pairs. The kernel and DTB
are single shared copies; paired root filesystems therefore do not provide
whole-system atomic A/B updating.

## Vendor GPT selection

Vendor attribute **bit 47** identifies selected/valid partitions. It is separate
from standard GPT legacy-bootable semantics. The 17.5.3 and early 18.21.10
snapshots had one valid GPT0 and erased GPT1; later experiments created valid
redundant records with different generations.

The current read-only decoder consumes `0x4400` bytes from each reserved GPT
area. Its accepted structure includes:

| Field | Checked value or meaning |
| --- | --- |
| Header within record | Offset 512, magic `EFI PART`, revision `0x10000`, size 92 |
| Primary current/alternate LBA | 1 / `0x3f7ff` |
| Backup current/alternate LBA | `0x3f7ff` / 1 |
| Usable LBA range | `0xc00` through `0x3f7de` |
| Entry array | Record offset 1024, 128 entries × 128 bytes |
| Entry-array LBA | Primary 2; backup accepts 2 or `0x3f7df` |
| Generation | LE32 at record offset 532, the vendor use of header offset 20 |
| Integrity | Header CRC with its CRC field zeroed, plus entry-array CRC |

These LBAs belong to the vendor's recorded format; their validation is not a
generic GPT layout inferred from nominal chip capacity. The decoder checks
exact kernel/DTB/root-bank ranges, names and selection. Exactly one root bank
must be selected, with kernel and DTB enabled. A non-erased invalid record is
rejected rather than silently ignored. If both valid records have equal
generation but different entry arrays, selection is ambiguous and rejected.
Otherwise the higher generation is selected. This implementation does not
establish a wrap-aware generation policy.

The kernel and custom updater share this decoder. The selected stock root is
exposed as read-only `vendor-rootfs`; the other root becomes writable `fpvos`.
The custom updater never changes GPT or stock-bank selection. Its older bank-0
assumption was superseded by this bank-aware behavior.

## Environment format

Each 384 KiB environment partition contains a **32 KiB environment at offset
zero**. The first four bytes are little-endian CRC32 over the following
`0x7ffc` bytes. It is not a five-byte redundant-environment header.

In the original 17.5.3 snapshot, both partitions were identical. Stored CRC
`0x567ea27a` matched the four-byte-header interpretation; the five-byte
interpretation produced `0x8548571f` and failed. The first eraseblock held the
32 KiB environment followed by 96 KiB zeros; two subsequent blocks were erased.
In 18.21.10, the logical environment still matched but partition tails differed,
including non-erased records of unresolved meaning. Those tails must be preserved.

U-Boot chooses `env0` or `env1` through vendor GPT bit 47. Standard upstream
two-entry redundancy instead changes the header layout and selection behavior,
so listing both copies in an ordinary redundant `fw_env.config` is incorrect.
The traced selected-copy importer falls back to built-in defaults on CRC failure;
the spare partition alone does not establish automatic retry.

A live read-only upstream `fw_printenv` test on 18.21.10 read all 21 variables
without CRC/default warnings using the selected partition and this geometry:

```text
# Historical env0-selected stock map; offsets are partition-relative.
/dev/mtd10ro 0x0 0x8000 0x20000 3
```

The tool was built from U-Boot v2020.10 commit
`050acee119b3757fee3bd128f55d720fdd9bb890`. The initial write exercise modified
only a disposable host file. Later October 25 work did persist `bootdelay=5`,
with valid CRC, unchanged partition tail and other verified boot partitions.
That write result alone did not prove a working UART prompt; prompt and command
authorization were established in subsequent work.

The active NAND route regenerates its boot command from boot-medium and GPT
state. Saved environment text containing `mmc read` or `ubi.mtd=15` is not an
authoritative account of the actual NAND boot, which used stock `ubi.mtd=16`
for userapp0 in the collected map. Editing `bootcmd` does not add an MMC driver
or establish an SD-first boot path.

## Restoration and rollback findings

A controlled October 28 test caused five unattended stock boots to panic before
userspace by selecting a missing init and a three-second panic restart. All five
kept `part_info=0x2baf`, userapp0 and unchanged GPT records. `bootcount`,
`bootlimit` and `altbootcmd` were absent before and after. No automatic bank
switch was observed in that scenario.

An alternate-root RAM-argument control stalled during SPI probing before root
mount, so it did not establish spare-bank bootability. The experiment does not
exclude watchdog-specific or undocumented update-state recovery.

Restoring environment variables through `saveenv` produced equivalent values
with different serialization order. Restoring the complete original environment
backup was needed for byte identity. The final protected-region readback matched
its pre-test snapshot, with valid environment CRCs and normal fpvOS NAND boot.

Later stock application boots changed some factory/data bytes in other studies.
Matching kernel/DTB or selected protected ranges must not be reported as
whole-device factory restoration. Raw NAND/OOB backups, ECC-main readbacks,
partition images and signed update containers are different artifacts.

See [USB RAM boot](gnd485-ram-boot.md), [stock update engine](ascent-stock-updater.md)
and [fpvOS installation/recovery](gnd485-fpvos-update.md).
