# P1 stock calibration and persistent storage

[Firmware](README.md) · [P1 Air](../devices/betafpv-p1-air.md)

The investigated P1 keeps per-unit radio power offsets in host NAND UBIFS
factory data, including `/factory/ar8030_pwr.json`. Factory occupies 6 MiB at
NAND offset `0x200000`. A generic firmware download cannot replace the unit's
calibration.

The recovered offset schema is `baseband.basic.power.pwr_offset`, containing
four integers in the studied −6 to +6 range. Stock merged configuration can
contain DEV-role defaults that the vendor application changes later; a saved
JSON file alone does not establish the active radio role.

This differs from the investigated Ascent VRX's separate 2 MiB radio SPI flash
and factory record at offset `0xa000`. Those storage arrangements must retain
their board scope.

The existing 6 MiB `usr_data` partition at `0x7100000` is separate from factory
and contains vendor `lowdelay` settings. Stock Linux can update UBI metadata
when mounting a volume, so raw partition hashes and calibration-file hashes
answer different preservation questions. The retained observations do not
establish absolute RF-power calibration accuracy or all stock settings formats.
