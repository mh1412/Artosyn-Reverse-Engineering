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

Video/control coexistence experiments use the hardware decoder and display while
custom MCU input processing and mixing run. Keep decoded-picture evidence
separate from command acknowledgements and stream counters.

## Custom MCU and EdgeTX work

Later development moved physical input acquisition, calibration and EdgeTX
mixing to custom MCU firmware at 500 Hz. Linux provides UI, configuration/model
storage, networking and video. The RF-off bench model passed Linux-reboot
independence and MCU reset/model restoration tests.

The October 26 scope tests located CRSF output on the external **RX/PPM**
contact. An optional active-drive build improved unloaded edges to about 47 ns
and a 3.16 V high level. Connected Nano tests still produced CRSF faults and
resets, with no received replies. A controlled module power cycle identified
the driver-fault → RC_IO → watchdog-reset path, also rebooting Linux.
See the [electrical experiments and their measurement limits](../experiments/g11-crsf-electrical.md)
and [half-duplex transport contract](../firmware/g11-crsf-transport.md).
Broader model timing, receiver delivery and remaining board services are still
unqualified.

The custom Linux/MCU session protocol is separate from the stock UART format.
See [protocol layers and formats](../protocols/layers-and-formats.md).

Detailed studies cover [startup, ADC/DMA acquisition and measured mixer timing](../soc/g11-gd32-acquisition.md)
and [memory qualification, segmented firmware and model commit recovery](../firmware/g11-mcu-memory-and-models.md).
The installed model-mirror layout and aligned commit-word fix supersede earlier
64 KiB/8 KiB envelopes and halfword-only commit assumptions.

Further studies document [power sequencing and the two custom shutdown designs](../firmware/g11-power-and-shutdown.md),
[battery/charging pins and stock warning thresholds](../soc/g11-battery-and-service-inputs.md),
and [MCU feedback with Linux speech presentation](../firmware/g11-feedback-and-audio.md).
Audio deployment and queue-drain tests do not establish audible output; the
complete autonomous off/wake sequence still needs physical qualification.
