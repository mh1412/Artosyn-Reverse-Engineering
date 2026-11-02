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

The [OTRA extraction study](otra-extraction.md) gives the normal container's
field layout, segment mapping, SPL/Sirius dispatch and the helper scripts'
authentication and extraction-completeness limits.

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

For any reproducible procedure, record the board revision, input hashes, component
versions, written ranges, readback result and final running state. A working
installation on one board is evidence for that combination, not a generic
installer for the whole chip family.
