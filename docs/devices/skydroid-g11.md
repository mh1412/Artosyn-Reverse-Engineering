# Skydroid G11

[Devices](README.md) · [Documentation index](../../README.md)

## Stock platform

The initial system investigation on 2026-10-22 examined G11 firmware 1.1.2,
build `20260714 15:57:53`. It identified a SigmaStar Linux host, an MCU control
interface, a separate SDIO Wi-Fi/Bluetooth device and an internally
USB-connected Artosyn AR8030 radio.

| Component | Observation |
| --- | --- |
| Host platform | DT model `INFINITY2M SSC011A-S01A-S`, compatible `sstar,infinity2m` |
| CPU | Two Cortex-A7 cores, 32-bit ARM hard-float userspace |
| OS | Vendor Linux 4.9.84, BusyBox 1.20.2, glibc 2.28 |
| Storage | NAND with UBI/UBIFS and separate configuration/application volumes |
| Control MCU | GD32F330-family; this unit passed restoring 128 KiB flash and 16 KiB SRAM tests |
| MCU connection | Stock application opens `ttyS1`; control UART observed at 115200 8N1 |
| Radio connection | Internal high-speed USB through the Artosyn driver |
| Display/video | Separate UI and `Videoplayer` processes |

Linux-visible memory is not total installed DRAM. The manufacturer manual's
memory specification and the device's reserved multimedia regions should be
kept separate from physical component identification.

## Control and radio paths

```text
Physical controls → MCU ↔ UART ↔ Linux application/HAL
                                      │
                          Artosyn client / host daemon
                                      │
                               kernel USB driver
                                      │
                                    AR8030
```

Ethernet is bridged to the radio network. Neither this bridge nor a connector
marked UART establishes that a CRSF transmitter interface is available.
The stock UI application, radio daemon, data forwarding process and video player
have different responsibilities and restart behavior.

## Video integration

The installed video player accepts Annex-B HEVC over UDP port 554. Its SDK
decoder setup is HEVC-specific; an H.264 stream is not interchangeable.
A local control socket selects playback modes. A successful mode response is
not proof that pictures were decoded, and a stale active flag can remain until
the player's no-data timeout is observed.

The [archived ingestion implementation](../systems/g11-stock-video-ingest.md)
documents UDP parser/cache boundaries and RTSP decoder reuse, queueing and
recovery limits separately from installed-player results.

## Stock studies

- [Linux platform, USB radio and connectors](g11-stock-platform.md)
- [GD32 identification and memory evidence](../soc/g11-gd32-memory.md)
- [Battery, charging and service inputs](../soc/g11-battery-and-service-inputs.md)
- [MCU images, trainer timing and feedback commands](../firmware/g11-stock-mcu-reconstruction.md)
- [MCU update protocol](../firmware/g11-stock-mcu-update.md)
- [Power signals and stock reset behavior](../firmware/g11-power-and-shutdown.md)
- [Stock MCU input formats](../protocols/g11-stock-inputs.md)
- [Video-control socket](../protocols/g11-stock-video-control.md)
- [Decoder and display lifecycle](../systems/g11-stock-display-lifecycle.md)
- [Vendor SDK services and limits](../systems/g11-vendor-sdk.md)
- [Stock PPM electrical observation](../experiments/g11-stock-ppm.md)
- [Vendor-player decoded-pixel validation](../experiments/g11-stock-video-validation.md)
