# Ascent Lite VTX / Sky482

[Devices](README.md) · [Documentation index](../../README.md)

## Identity and architecture

The investigated unit identified as `Ascent_lite`, board
`FPV-Ascent-Sky-482-V1.3-1.1`, with `Ascent_H_Sky_18_21_10` before the startup
incident investigated on 2026-10-16.

Matched 17.5.8 H Sky/Lite firmware analysis shows an ARMv7/musl Linux host with
a Hi3516CV610 SDK profile, `ar_ldyhs_sky` backend, `daemon_sdiov12` and a separate
AR8030-family SDIO radio. The host handles the camera/media pipeline; the radio
runs SKY firmware and board configuration. SDK/profile evidence does not by
itself identify the physical host chip package or establish its live clock rate.

| Interface or component | Evidence |
| --- | --- |
| Normal USB | `1d76:0101`, SerialGadget / CDC ACM |
| Recovery USB | `3361:d001`, USBBurn, observed during bind/update-button power-up |
| NAND | Electronic ID `EF AA 21`, confirming Winbond W25N01GV family |
| Package marking | Owner-reported marking interpreted as W25N01GVZEIG |
| NAND organization | 128 MiB data capacity; 2,048-byte pages plus 64-byte spare area |

The management serial interface should not be assumed to be a Linux shell.
USB enumeration also does not prove that Linux, DDR or the radio has booted.

After repair, authenticated live access confirmed Linux 5.10.221, bank selector
`fpvbsel=0`, and `ttyAMA0` at 115200 with a running getty. The boot command line
allocated `mem=64m`; that is not a physical DRAM capacity measurement. Kernel
logs selected the `fpv405_qfn` board configuration and `cv2004` sensor configuration,
which identify software selections rather than independently read package markings.

## Startup failure and repair

The VTX stopped exposing normal USB and its radio link after script-upload
experiments. Two complete raw NAND reads at different SPI rates matched.
Reconstruction of the HiSilicon controller page layout allowed the application
filesystem to be mounted with journal replay for diagnosis.

Two helpers, `temp_measure.sh` and `config_ether.sh`, were zero bytes while
retaining their executable modes and ownership. The stock startup script waited
for a numeric result from the empty temperature helper, establishing a concrete
startup blocker.

The repair restored those scripts. A separate authorized account change replaced
only the root password hash. The reconstructed encoder reproduced all 24,576
original pages of the affected partitions, including controller ECC and spare
metadata, before the repair was written.

The combined repair affected 37 erase blocks. Preimages were checked before
writing, blocks were verified afterward, and a complete 132 MiB raw readback
matched the prepared candidate. The extra raw size includes spare bytes;
it is not additional NAND data capacity.

Normal boot and USB returned after the flash connection was resoldered. The
device answered its stock identity query, USB Ethernet switching worked, root
login succeeded, and both repaired scripts matched stock hashes and modes.
The stock radio link resumed and an HDMI capture showed the camera scene with
receiver overlays.

This establishes successful storage repair and visible stock video for this
unit. It does not establish long-run reliability or validate a replacement AP
radio image.

The [complete experiment](../experiments/ascent-lite-flash-repair.md) preserves
the failed upload sequence, exact write boundary, readback hashes and runtime
checks. The [NAND study](../firmware/sky482-nand-layout.md) documents raw-mode
capture, partition offsets, reader framing and the recovered controller layout.

## Recovery format distinction

The [Kestrel Air study](../systems/kestrel-air.md) covers the replacement air
application, vendor-library boundary, slice streaming, intra refresh and
optional camera IMU metadata. Its separate P1 backend has different capabilities.

The vendor update `.img` is an ASW update container, not a raw NAND image or a
complete BootROM recovery kit. The observed USBBurn interface and a matching
tool profile are insufficient to establish a working general recovery procedure.

See [firmware formats and update paths](../firmware/architecture-and-updates.md)
and the [receiver study](ascent-vrx.md) for the other side of the system.
The [17.5.8 air-unit updater trace](../firmware/ascent-air-unit-updater.md)
details ASW descriptors, validation and bank writes. The
[air/ground comparison](../systems/ascent-air-ground-comparison.md) separates
shared radio interfaces from incompatible host ABIs and unverified role conversion.
