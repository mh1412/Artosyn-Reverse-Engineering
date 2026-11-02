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

The 17.5.3 architecture capture was taken after access modifications: a password
startup wrapper and an explicitly selected 17.5.8 continuation script were in
the boot path. It therefore records a modified running system, not a pristine
17.5.3 startup trace.

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

This establishes a useful ground-station interface without receiver-side
decode/re-encode. A later proxy/export application is a separate implementation
step from this passive extraction experiment.

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
