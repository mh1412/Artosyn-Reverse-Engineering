# BETAFPV P1 Air

[Devices](README.md) · [Documentation index](../../README.md)

This study covers the standard P1 Air unit investigated from October 22 through
November 1/2 UTC, 2026, initially running **P1_SKY 1.0.44.rel**. It does not describe
the P1 AIO or Matrix. The tested camera is **SC231AI**, distinct from the
NT99235 used by the related VR04-air reference implementation.

## Hardware and stock software

| Component | Recorded result |
| --- | --- |
| Application SoC | Artosyn AR9311; live DT `artosyn,proxima-9311` |
| CPU | Two online Cortex-A53 cores, AArch64 |
| Stock software | Linux 4.9.38, glibc 2.25, P1_SKY 1.0.44.rel |
| Memory | Live 256 MiB span at `0x20000000`; 128 MiB Linux and 128 MiB media |
| NAND | XTX 128 MiB SPI NAND, 128 KiB erase blocks, 2,048-byte pages, 128-byte OOB |
| Radio | AR8030-family SDIO device, running ID `4152:8031` |
| Camera | SC231AI, chip ID bytes `cb 6a`, I2C address `0x30` |
| Stock USB | `1d6b:0101` Artosyn Sirius, CDC serial and RNDIS networking |
| Console configuration | `ttyS0`, 1,152,000 baud; physical UART pin mapping unverified |

The stored official device tree declares a generic 1 GiB range; this is not the
measured board memory layout. Linux reported 113,052 KiB usable memory, a 16 MiB
CMA allocation and a separate 128 MiB vendor MMZ beginning at `0x28000000`.

The October 23 stock-boot check independently confirmed both CPUs online, each
with MIDR `0x410fd034` (Cortex-A53 r0p4), and a reported 800 MHz frequency. Both
installed DTB partitions held identical 18,416-byte trees with SHA-256
`4c7bd54c4e5f0d709a054702bc410b9df709135e4335af17f2d1e297f0f15dfc`.
Stored/live tree differences were limited to boot arguments and memory range.
These establish the AR9311 platform; MIDR identifies the ARM core, not an Artosyn
SKU. An independently inspected chip marking was not part of that verification.
The downloaded 2.1.2 update did not substitute for installed 1.0.44 evidence.

Stock camera processing runs through `ar_lowdelay -m 2 -t 1`, the vendor MPP
libraries, MMZ and kernel modules. No stock `/dev/video*` devices were found.

## Board wiring

Stock startup identifies radio reset as GPIO23, active low. The stock device
tree identifies bind input GPIO42, active low. These are controller GPIO numbers,
not connector pin numbers. Physical long-press behavior and a complete stock
camera/LED pin map need separate confirmation.

## Firmware comparison

The investigated official v2.1.2 image was 23,646,748 bytes with SHA-256:

```text
739124ded12826c6ff5e0d70ed0266810975e0071d46980a37c4383f35778669
```

Its OTRA body digest and UBI header CRCs passed extraction checks; the vendor
RSA signature was not verified. This is a measured file identity, not a vendor
published checksum or a recommendation to upgrade.

The two codec blobs match between the installed 1.0.44 and inspected 2.1.2
images. The sensor library, ISP tuning, media library, kernel modules and radio
configuration differ. A shared codec blob therefore does not establish matching
camera behavior or radio compatibility across the releases.

## Related stock studies

- [SC231AI camera and ISP findings](../soc/ar9311-p1-camera.md)
- [Calibration and persistent storage](../firmware/p1-calibration-and-settings.md)
- [Boot and NAND findings](../firmware/p1-boot-and-recovery.md)
