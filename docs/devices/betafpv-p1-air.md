# BETAFPV P1 Air

[Devices](README.md) · [Documentation index](../../README.md)

This study covers the standard P1 Air unit investigated on 2026-10-31 and
2026-11-01/02 UTC, initially running **P1_SKY 1.0.44.rel**. It does not describe
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

Stock camera processing runs through `ar_lowdelay -m 2 -t 1`, the vendor MPP
libraries, MMZ and kernel modules. No stock `/dev/video*` devices were found.
The source replacement uses a different, V4L2-based architecture.

## Board wiring

The following assignments are established by stock evidence and the tested
source device tree. These are controller GPIO numbers, not connector pin numbers.

| Function | Assignment | Evidence/behavior |
| --- | --- | --- |
| Radio reset | GPIO23, active low | Stock startup and native SDIO bring-up |
| Camera reset | GPIO107, active low | Source sensor initialization on the tested board |
| Camera power-down | GPIO104, active high in the source binding | Source sensor initialization |
| Camera clock | 24 MHz | Tested SC231AI mode configuration |
| CSI connection | Two data lanes, 378 MHz link frequency | Source DT for 1920×1080/60 RAW10 mode |
| Bind input | GPIO42, active low | Stock DT and physical press/release checks |
| Red status LED | GPIO0, active low | Physical color/polarity observation |
| Green status LED | GPIO1, active low | Physical color/polarity observation |

Bind input is polled every 20 ms with 20 ms debounce and exposed as
`KEY_WPS_BUTTON` on `p1-air-bind`. The source application discovers the device by
name and reads current key state, rather than relying on an event number.

A release starts a 30-second pairing window. An observed 11.642-second hold
caused pairing on release without upgrade or reboot. Red indicates pairing;
green blinks when disconnected and remains steady, with red off, during video.
These are source-application semantics; they are not a promise of identical
stock long-press behavior.

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

## Source replacement status

The source stack demonstrated camera capture, HEVC encoding, automatic exposure
and white balance, radio transport, physical pairing/LEDs, persistent settings
and supervised application startup. Later work directly booted Linux 6.18.36
from NAND bank 0 while retaining stock bank 1.

The direct-boot validation completed two cold boots, native clean-mode update,
stock return and a watchdog-to-USB recovery test. It supersedes the earlier
RAM-only development stage. This remains a tested development implementation;
its results do not establish flight/range performance or full vendor parity.

- [Camera, ISP and encoder](../soc/ar9311-p1-camera.md)
- [Radio calibration and persistent state](../firmware/p1-calibration-and-settings.md)
- [Direct boot, update and recovery](../firmware/p1-boot-and-recovery.md)
- [Validation sequence and limitations](../experiments/p1-source-stack.md)
