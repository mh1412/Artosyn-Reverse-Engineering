# Ascent Lite recovery references and unresolved boot stages

[Firmware](README.md) · [Ascent Lite](../devices/ascent-lite-vtx.md)

The October 16 recovery investigation retained HiSilicon BurnTool/ToolPlatform
guides, a Hi3516CV610 SDK overview and an OpenIPC Defib overview. They explain
possible recovery mechanisms, but none is a completed USB recovery procedure
for the investigated Sky482. The demonstrated repair used
[raw NAND reconstruction and readback](../experiments/ascent-lite-flash-repair.md).

## Evidence for the investigated unit

| Observation or reference | What it establishes | What remains separate |
| --- | --- | --- |
| `3361:d001` / `USBBurn` during update-button power-up | A recovery USB interface enumerated | DDR initialization, loader execution and flash programming |
| CV610 SDK profile in the matched stock application | A relevant software/API family for investigation | Physical package identification and a matching recovery loader |
| BurnTool lists Hi3516CV610 | The retained tool documentation describes that target family | Sky482 DDR parameters, flash geometry, board wiring and accepted payloads |
| Successful reconstructed NAND repair | Exact affected blocks were written and verified, then stock video returned | General BootROM recovery from an arbitrary damaged image |

The stock ASW updater, BootROM recovery and external raw NAND programming have
different input formats and write boundaries. A vendor update `.img` is not
automatically suitable for the latter two paths.

## BurnTool's documented boot chain

The retained BurnTool guide is revision 00B02, dated September 30, 2026; the
ToolPlatform guide is 00B01, dated June 30. Its documented sequence first
exchanges DDR parameters with BootROM, initializes DDR, transfers U-Boot into
DDR and executes it. Only afterward does the tool use the running loader's
commands to program flash. The guide places DDR setup near the 5% point of
the U-Boot download display and completion of transfer at 100%; neither UI
percentage alone is a verified flash write.

For other partitions, the guide distinguishes loading a bootloader on a board
without one from using an existing bootloader. Network transfer uses TFTP and
loader write commands. USB and serial are additional documented transports;
their availability does not remove the need for a matched loader and DDR setup.
ToolPlatform selects a chip solution and disables tools that do not support it.
Selecting a solution in the UI does not identify the connected board.

The CV610 compatibility table lists SPI NOR, SPI NAND/NAND, eMMC and serial,
network and USB interfaces. This is tool-family coverage, not a claim that the
Ascent Lite exposes all those interfaces. The same manual's separate DDR-only
burn tab is explicitly scoped to **Hi3511CV100 USB slave boot**. Its fixed
`0x41000000`/`0x42000000` address rules must not be imported as a CV610 or
Sky482 RAM-loading recipe.

## Partition and readback semantics

Several details matter when interpreting a BurnTool session:

- The XML partition list controls the tool's write plan; the guide says it does
  not determine the board's actual boot partition layout. It must agree with
  the bootloader's partition configuration.
- A selected partition with no image file is an **erase request**, not a skipped
  partition. Start addresses and lengths must meet the relevant flash alignment.
- NAND readback distinguishes data with and without OOB. The guide uses
  data-only reads for boot/kernel/UBIFS and data-plus-OOB for YAFFS; this does
  not identify Sky482's reconstructed controller layout automatically.
- `pure data length` is payload length, while `len_incl_bad` includes space
  occupied by skipped bad blocks. Neither includes OOB size.
- Serial readback is not supported by the guide's address-upload workflow;
  network readback needs a loader with `tftpput`. Its eMMC upload workflow is
  network-only.
- Optional TFTP CRC checking compares transferred data with a board-reported
  CRC32. It is not a vendor signature check or proof of successful reboot.

The troubleshooting guide separates start-frame failure, head-frame failure,
data-frame failure and loader-command failure. It associates these with
different possible boundaries: boot-entry timing/serial selection, mismatched
boot image or DDR initialization, transfer interruption, and selected flash
type. These are diagnostic leads, not diagnoses of the Ascent failure.

The manuals include GUI screenshots, host setup advice and historical Java/USB
driver dependencies. This review used their textual descriptions; it did not
execute the tools, inspect the referenced screenshots, or qualify a current
host installation.

## What the public CV610 SDK overview contributes

The retained SDK overview calls itself a beta and describes a `dmeb` development
board build, with separate board configuration, media drivers/libraries,
toolchain, samples and ToolPlatform components. A default build does not
automatically place the media modules and libraries in the root filesystem.
Its `pack` option adds them, installs `sdk_server` and a sample application,
and starts the media service during boot.

Its named Sunniwell development kit uses a Hi3516CRNCV610 profile with two
Cortex-A7 cores at 950 MHz and 128 MiB integrated DDR3. Those are **reference
board specifications**, not measurements of the Ascent Lite. The overview's
default sensor is HY006-3814-0011; the investigated Lite's kernel selected
CV2004. A shared SDK name does not make the demo image a board port.

The SDK overview also separates source components from binary-library terms.
Its availability does not establish a completely source-built media runtime or
a reusable Ascent recovery bundle.

## Defib is a separate recovery lead

The retained Defib README describes three HiSilicon/Goke UART protocol families:
Standard, V500 and CV6xx. Its U-Boot-based recovery path initializes the target,
loads a bootloader into RAM, detects flash and transfers/programs partitions
through that loader. A separate bare-metal agent uses COBS-framed serial traffic
at 921600 baud for flash operations. Those are two distinct implementations.

The same README's named physical examples are CV300/EV300 camera boards. Its
QEMU example covers the Standard protocol and explicitly excludes V500/CV6xx
emulation at that checkpoint. A broad chip-support claim, a web serial UI or a
passing Standard-protocol emulation is therefore not CV610/Sky482 recovery
evidence. Automatic OpenIPC layouts and environment migration likewise cannot
substitute for the Ascent's own flash map and calibration data.

No Defib loader, agent, power-control backend, emulator or flash operation was
run during this review. The unresolved recovery boundary remains a matched
Sky482 loader/DDR configuration followed by demonstrated transfer, readback
and stock boot.
