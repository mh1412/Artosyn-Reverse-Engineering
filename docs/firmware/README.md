# Firmware

[Documentation index](../../README.md)

Stock image formats, boot chains, binary reconstruction, updates and recovery.

## Overview and image formats

- [Firmware components, formats and update paths](architecture-and-updates.md)
- [OTRA containers and extraction-tool boundaries](otra-extraction.md)

## AR8030 radio

- [AR8030 loading, transports and driver lifecycle](ar8030-loading-and-driver.md)
- [AR8030 boot, worker reconstruction and host lifecycle](ar8030-boot-and-lifecycle.md)
- [AR8030 stock configuration and board boundaries](ar8030-configuration.md)
- [AR8030 stock factory records and flash access](ar8030-factory-storage.md)
- [AR8030 internal service construction](ar8030-internal-services.md)
- [AR8030 command construction and physical frame duration](ar8030-command-engine.md)
- [AR8030 transmit scheduling and stream ownership](ar8030-tx-and-stream-ownership.md)
- [AR8030 power-control reconstruction](ar8030-power-control.md)

## Ascent access and updates

- [Ascent stock root access and account overlays](ascent-root-access.md)
- [Ascent stock startup and one-shot maintenance hooks](ascent-startup-hooks.md)
- [Ascent stock update engine and authentication](ascent-stock-updater.md)
- [Ascent Lite ASW update engine and bank selection](ascent-air-unit-updater.md)
- [Ascent PC-tool dispatch and cross-model update boundaries](ascent-update-compatibility.md)

## Ascent storage and recovery

- [Gnd485 NAND layout, GPT and environment](gnd485-nand-and-gpt.md)
- [Gnd485 U-Boot access and USB RAM loading](gnd485-ram-boot.md)
- [Sky482 NAND capture and controller page layout](sky482-nand-layout.md)
- [Ascent Lite recovery references and unresolved boot stages](ascent-lite-recovery-references.md)

## BETAFPV

- [P1 stock boot and NAND findings](p1-boot-and-recovery.md)
- [P1 stock calibration and persistent storage](p1-calibration-and-settings.md)
- [Reference P1_SKY stock access and package boundaries](vr04-stock-access.md)

## G11 companion MCU

- [G11 stock MCU images, trainer timing and feedback commands](g11-stock-mcu-reconstruction.md)
- [G11 stock MCU update protocol](g11-stock-mcu-update.md)
- [G11 stock power signals and reset behavior](g11-power-and-shutdown.md)

## AR9311 codec

- [Wave521C codec firmware reconstruction](wave521c-reconstruction.md)
