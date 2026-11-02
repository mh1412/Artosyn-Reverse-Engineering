# Ascent VRX / Gnd485

[Devices](README.md) · [Documentation index](../../README.md)

## Identity and scope

The standard Ascent VRX uses an Artosyn Proxima-9311 application SoC and a
separate AR8030-family radio/baseband device. This study concerns Gnd485, not
the RK3568-based VRX Pro.

The initial receiver identified as `Ascent_VRX`, board
`FPV-Ascent-Gnd-485-V1.2-1.0`, running `Ascent_G_Gnd_17_5_3`.
Architecture observations were collected on 2026-10-05. Later work used
18.21.10; results from those versions must retain their individual scope.

## Platform

| Component | Recorded observation |
| --- | --- |
| Application SoC | Live DT compatible `artosyn,proxima-9311` |
| CPU | Two Cortex-A53 cores; observed 800 MHz with performance governor |
| Stock OS | Linux 4.9.38, AArch64, BusyBox 1.25.0 and glibc 2.25 |
| Memory | Live DT describes 512 MiB; 256 MiB Linux region and 256 MiB vendor media region |
| Storage | Driver reports 128 MiB GigaDevice SPI NAND |
| Radio | Separate AR8030-family device; running SDIO ID `4152:8031` |
| Radio host driver | `sdio_artosyn` |
| USB | Management serial and USB networking functions |

The memory result describes the live configuration and allocation, not a
destructive physical DRAM capacity test. An earlier reference DT described a
different memory range and should not override the live board observation.

## Software architecture

BusyBox init starts the filesystem and application scripts. The vendor stack
includes radio firmware loading, `daemon_sdio`, the `ar_ldy_gnd` backend,
management services and a Qt framebuffer UI. Clean-mode selection provides a
separate maintenance/update path.

The detailed [stock service architecture](../systems/ascent-stock-architecture.md)
maps process/socket ownership, Binder/CUSE media services, memory and persistent
mounts. `/tmp/fpvdata` is UBIFS-backed persistent storage despite its pathname;
it differs from the RAM copy of the USB updater.

The 17.5.3 architecture capture was taken after access modifications: a password
startup wrapper and an explicitly selected 17.5.8 continuation script were in
the boot path. It therefore records a modified running system, not a pristine
17.5.3 startup trace.

The [access study](../firmware/ascent-root-access.md) distinguishes the uninstalled
rootfs patch, the 17.5.3 startup wrapper and later reboot-verified 18.21.10
passwd/shadow overlays. [Startup hooks](../firmware/ascent-startup-hooks.md)
document the tested one-shot clean-mode route and its normal-boot boundaries.

## Compressed-video extraction

A passive capture of the receiver's existing loopback TCP session exposed
compressed video before the receiver decoded it:

```text
VTX encoder → RF → daemon_sdio → TCP 50000 → ar_ldy_gnd
                                      │
                                passive copy
                                      ↓
                        TCP reassembly and deframing
                                      ↓
                             HEVC → host decoder
```

The experiment decoded 316 frames of 1920×1080 HEVC Main video on the host.
Vendor metadata reported 60 fps; the sample was remuxed at that rate without
re-encoding. Its raw elementary stream did not supply container timestamps,
so the playback rate alone is not a measured RF cadence or latency result.

The later [ground-station implementation](../systems/ascent-groundstation.md)
exported this stream through a transparent TCP proxy and direct HEVC RTP output.
Persistent startup and subsequent receiver/host decoding were verified after
reboots. That work also records the failed process-only restart and an incident
where a connected flag remained true despite no incoming video.

## Debug UART

The initial [UART0 investigation](../experiments/ascent-uart0.md) found that the
original CP2102 adapter clamped a requested 1,152,000 baud to 1,000,000. No
tested rate produced readable console text. A boot-burst scope measurement
supported approximately 1.15 Mbaud and likely 1.8 V signaling, conditional on
probe attenuation. Post-boot adapter garbage was not established console traffic.

## Flash identification

The reported marking was GigaDevice, `AJ2507`, `5F1GM7RFWIG`. The original
lookup identified the GD5F1GM7R family but did not resolve the RF ordering suffix.
The lot/date interpretation of `AJ2507` was not verified.

The recorded GD5F1GM7RE family reference specifies 1 Gbit main data, nominal
1.8 V (1.7–2.0 V), SPI x1/x2/x4 and 2-KiB pages. Its Linux definition uses
manufacturer/device ID `c8 81`, 128-byte OOB, 64 pages per block and 1,024 blocks:
128 MiB main data or 136 MiB with all OOB. These family facts do not establish
the exact package/pinout or programmer configuration for the unresolved marking.

The saved live boot log adds evidence beyond the original marking-only study:
it identifies GigaDevice SPI NAND, reports 128 MiB / 128-KiB erase / 2,048-byte
page / 128-byte OOB, and logs repeated `c8 81` ID bytes. The same probe sequence
first reports a failed NOR probe and all-zero intermediate NAND ID, then changes
B0 from `0x11` to `0x51` before another ID-like read. These lines require their
operation context; they are not all independent part identifications. This is
driver-log evidence, not a standalone programmer read or exact RF-suffix proof.

Flash-family supply voltage and UART voltage are separate evidence. Supply-only
level conversion does not establish matching clock/data I/O levels. The
[Sky482 Winbond repair](../firmware/sky482-nand-layout.md) concerns a different
chip/controller layout. The [Gnd485 NAND study](../firmware/gnd485-nand-and-gpt.md)
distinguishes logical MTD readback from an OOB-inclusive physical dump.

## Related studies

- [AR9311 CPU, media and platform](../soc/ar9311.md)
- [AR8030 radio and Open8030](../soc/ar8030.md)
- [System video paths](../systems/video-paths.md)
- [Firmware and update boundaries](../firmware/architecture-and-updates.md)
- [NAND/GPT layout and environment](../firmware/gnd485-nand-and-gpt.md)
- [U-Boot access and source-kernel RAM boot](../firmware/gnd485-ram-boot.md)
- [fpvOS installation and vendor restoration](../firmware/gnd485-fpvos-update.md)
- [Native SDIO radio bring-up](../experiments/gnd485-sdio-radio.md)
- [Shared-DMA decode and HDMI output](../systems/ar9311-display-pipeline.md)
- [Stock backend commands and telemetry fields](../protocols/ascent-backend-control.md)
- [18.21.10 serial/MSP forwarding](../protocols/ascent-stock-msp.md)
- [Kestrel software overlay and backend ownership](../systems/kestrel-software-ui.md)
- [Live HDMI/UI and worker-recovery validation](../experiments/kestrel-ascent-ui.md)
- [Open-stack comparison and later AR9311 port](../systems/open-receiver-stacks.md)
