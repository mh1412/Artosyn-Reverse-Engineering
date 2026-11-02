# Firmware

[Documentation index](../../README.md)

## Articles

- [Firmware components, formats and update paths](architecture-and-updates.md)
- [Gnd485 NAND, GPT selection and U-Boot environment](gnd485-nand-and-gpt.md)
- [Gnd485 U-Boot access and USB RAM boot](gnd485-ram-boot.md)
- [Ascent stock update engine and authentication](ascent-stock-updater.md)
- [Gnd485 fpvOS installation, clean recovery and vendor restoration](gnd485-fpvos-update.md)
- [Wave521C codec firmware, partial ISA and diagnostic patch](wave521c-reconstruction.md)
- [AR8030 and Open8030](../soc/ar8030.md)
- [AR8030 image loading, USB/SDIO and driver lifecycle](ar8030-loading-and-driver.md)
- [AR8030 configuration and board-profile boundaries](ar8030-configuration.md)
- [AR8030 calibration caches, power commands and failed dynamic integration](ar8030-power-control.md)
- [Open8030 runtime, queues, diagnostics and session behavior](open8030-runtime.md)
- [Ascent Lite startup failure and repair](../devices/ascent-lite-vtx.md#startup-failure-and-repair)
- [P1 direct NAND boot, update and recovery](p1-boot-and-recovery.md)
- [P1 calibration, persistent settings and supervision](p1-calibration-and-settings.md)

## Components to keep separate

| Component | Function |
| --- | --- |
| Application-host firmware | Linux kernel, device tree, root filesystem, services and UI |
| AR8030 radio firmware | Host transport, RF/baseband control and radio payload handling |
| Wave521C codec firmware | Codec processor execution alongside the host-side media driver |
| Controller MCU firmware | Physical input acquisition, control/mixing and board management |

A combined update package can contain several of these components. Record their
individual hashes, load destinations, board roles and tested combinations.
Shared symbol names or container formats do not prove binary compatibility.
