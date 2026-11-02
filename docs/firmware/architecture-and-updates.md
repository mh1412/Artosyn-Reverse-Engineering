# Firmware components, formats and update paths

[Firmware](README.md) · [Documentation index](../../README.md)

## Firmware layers

The application host, AR8030 radio, Wave521C codec processor and controller MCU
have separate execution environments. A vendor package can distribute several
components together, but each needs its own identity and compatibility scope.

| Layer | Examples of what must be identified |
| --- | --- |
| Boot chain | Bootloader/secure firmware, environment, partition selection |
| Linux host | Kernel, DTB, root/application filesystems, startup scripts |
| Radio | Bootstrap, runtime, board profile, SKY/GND or AP/DEV configuration |
| Codec | Codec CPU firmware and the matching host-driver interface |
| MCU | Board initialization, input/control firmware and Linux protocol |

## Containers and flash images

ARTO/OTRA extraction tooling describes package fields and nested components.
The Ascent Lite stock update is described as an ASW container. These labels and
formats require image-specific parsing; neither an update extension nor a shared
tool family makes an image a raw flash dump.

A raw NAND capture includes page organization, spare bytes, controller ECC and
bad-block information that an update container may not contain. The Sky482 repair
required reconstruction of that controller layout, not simply writing the vendor
update file at address zero.

The [Sky482 NAND study](sky482-nand-layout.md) gives the observed byte mapping
and BCH parameters. The [repair record](../experiments/ascent-lite-flash-repair.md)
distinguishes a matched storage candidate from verified boot and stock video.

## Ascent boot and update behavior

The standard VRX stock stack boots Linux and runs BusyBox/vendor startup scripts,
mounts persistent storage and selects normal or clean-mode startup. Management
and update software then handles staged payloads and version-specific install
operations. The PC-tool upload protocol and the target's update procedure are
different layers of the same workflow.

Detailed studies cover the [USB wire protocol](../protocols/ascent-usb-management.md),
[stock engine and authentication](ascent-stock-updater.md),
[NAND/GPT/environment](gnd485-nand-and-gpt.md) and verified
[U-Boot USB RAM loading](gnd485-ram-boot.md).

An update comparison must identify the model dispatch, destination component,
partition selection and actual target-side writer. Shared updater symbol names
do not prove that all receivers or air units accept the same procedure.

The [H Sky 17.5.8 updater](ascent-air-unit-updater.md) provides a concrete
comparison: a 128-byte ASW envelope, transfer MD5 and payload CRC in its traced
user-space path, paired root/application writes, and shared boot/environment
writes. Its function named `unpack_images_mmap` programs flash during iteration.
This differs from the receiver's image-authentication and GPT selection logic.

## Two AR9311 fpvOS updater designs

The research includes distinct custom-kernel and stock-kernel paths:

| Design | Recorded behavior and boundary |
| --- | --- |
| Custom-kernel updater | Updates shared kernel/DTB and stages the root bundle in the inactive application partition; includes a recovery initramfs and verified restoration inputs |
| Stock-kernel updater | Retains the actual vendor kernel/DTB and stock application partition, installs fpvOS in the alternate application partition, and uses GPT selection to choose the root |

Both use RAM execution to avoid updating a mounted root in place, but their
failure boundaries differ. The custom-kernel design's shared kernel/DTB write
was not atomic against power loss; RAM rollback after a reported error cannot
cover every interruption during programming.

The stock-kernel design validated USB installation, NAND readback, stock/fpvOS
selection and unattended boot. Its fault tests included simulated power-cut
points, not deliberate interruptions during real flash writes. Stock boots can
modify factory/application data, so matching protected partition hashes does not
mean the entire device was restored byte-for-byte.

The [Gnd485 installation/recovery contract](gnd485-fpvos-update.md) records
the current bank-aware map, bundle/control formats, transaction ordering and
vendor restoration boundaries.

## Replacement firmware scope

The separate [P1 direct-boot design](p1-boot-and-recovery.md) uses the P1_SKY
1.0.44 geometry, source kernel/DTB/application bank 0 and preserved stock bank 1.
It has its own independent recovery and activation sequence. Neither Ascent
receiver updater layout should be applied to P1.

[Open8030](../soc/ar8030.md) replaces the radio firmware, while fpvOS work addresses
the application host. Neither automatically replaces the Wave521C codec firmware.
The G11 custom MCU is another independent replacement with its own host protocol.

For any reproducible procedure, record the board revision, input hashes, component
versions, written ranges, readback result and final running state. A working
installation on one board is evidence for that combination, not a generic
installer for the whole chip family.
