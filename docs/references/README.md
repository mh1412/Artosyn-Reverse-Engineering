# Public references and their research scope

[Documentation index](../../README.md)

These references were used in the recorded investigations. Pinned revisions
identify historical research inputs, not the latest upstream releases. A link
does not establish compatibility with every board in this wiki.

## Firmware and protocol reconstruction

| Reference | Recorded contribution and boundary |
| --- | --- |
| [ar-firmware-tools](https://github.com/fpv-wtf/ar-firmware-tools/tree/74a9c3573388660c2600c0d2ecc4045ce0698278) | OTRA image definitions; distinct from the similarly named USB serial frame. |
| [Ascent extractor](https://github.com/umeow0716/extract-ascent-otra/tree/ca85dbc8f63af990916754f858f9f337fbdc704d) | Extracted public 16.5.7 firmware used to reconstruct the information query. Extraction is not installation or live-image identification. |
| [AR8030 Go client](https://github.com/crosstyan/ar8030-go-client/tree/0a68cdb7e17b32730cbfd9cc88f8c734b8f7b3df) | Host frames/getters. The initial failed probe used a wrong local XOR initializer; the reference already used `0xff`. |
| [AR8030 transport](https://github.com/henkwiedig/ar8030-transport/tree/bb6e14f1f0952f85aa5ab6ded14d3782cf6dd372) | SDIO container/loader and non-IP baseband sockets, not an interchangeable application image. |
| [FreeArtLynk](https://github.com/henkwiedig/FreeArtLynk/tree/29b900abad732c33283070c419f67e39b37c31c9) | Separate U-Boot HID format and transmitter encoder/RTP architecture. Flasher not executed in the initial study; no Sky482 receiver-conversion proof. |
| [walksnail-root](https://github.com/aj22r/walksnail-root/tree/7f5a809c77f97522bb341c253b66c43fce72a8e2) | Avatar Goggles X debug-rooting reference; endpoint, protocol and memory map differ from Ascent. |
| [U-Boot v2020.10](https://github.com/u-boot/u-boot/tree/050acee119b3757fee3bd128f55d720fdd9bb890) | NAND-aware environment reader. Live reads and disposable host-file edit tests did not install new U-Boot firmware. |

Related studies: [USB management](../protocols/ascent-usb-management.md),
[radio framing](../protocols/ar8030-host-link.md),
[update compatibility](../firmware/ascent-update-compatibility.md),
[root access](../firmware/ascent-root-access.md), and
[NAND/environment layout](../firmware/gnd485-nand-and-gpt.md).

## Platform and application references

| Reference | Recorded contribution and boundary |
| --- | --- |
| [MissingLynk wrapper](https://github.com/Missing-Lynk/MissingLynk/tree/e08dcb0e32c0150f79bc9e641bc62e8e2494e983) | Early device-profile, USB/recovery and build documentation; BetaFPV defaults do not automatically apply to Ascent. |
| [MissingLynk kernel](https://github.com/Missing-Lynk/ml-kernel/tree/e3885c6394a7025eacd3a23f473276d9823beb25) | Retained AR9311 platform drivers, NT99235 camera/ISP reconstruction and reference-board validation/gaps. |
| [MissingLynk userspace](https://github.com/Missing-Lynk/ml-userspace/tree/674ec00ecde0bba2eb844def1b4a0b7f8e685e13) | Retained GStreamer, radio/session control and HUD; tiled reference video differs from the captured single-stream Ascent mode. |
| [MissingLynk rootfs](https://github.com/Missing-Lynk/ml-rootfs) | Alpine profiles and required radio/codec/tuning inputs. |
| [Unofficial Proxima reference](https://github.com/Missing-Lynk/ml-datasheets/tree/91c9ac90d0e38ed02a6a8ae172986df5ac6c03a3) | Register/platform reconstruction, rechecked October 7; not a vendor datasheet or Gnd485 physical-capacity measurement. |
| [fpvOS](https://github.com/gehee/fpvOS) | Original RK3568 application work and later AR9311 receiver/air port. Early “Pro only” assessments are historical. |
| [Kestrel UI assessment baseline](https://github.com/gehee/kestrel-gnd/tree/e60d59e4618448040ea35e54b0e2cd4244a4157e) | Original GLES coupling inspected before standalone LVGL work. |
| [VR04 video injector](https://github.com/XuliGan4eg2006/VR04_HD_video_injector) | BetaFPV-specific raw-video injection; its documented password was rejected by the studied Ascent receiver. |
| [BetaFPV P1 VideoCast](https://github.com/hermanmitish/BetaFPV-P1-VideoCast) | Model-specific H.265/OSD tee and desktop capture. |
| [beta-dvr](https://github.com/harrismcc/beta-dvr) | Browser WebUSB/MTP DVR reference; inspected stock Ascent had no MTP interface. |
| [Artosyn thermal notes](https://github.com/lat-gaurav/thermal/tree/d10ded2e3526663d1fb5c30e923797cd44e011b9) | Sirius USB naming is shared; the thermal protocol differs from Ascent OTRA. |

See [open receiver stacks](../systems/open-receiver-stacks.md),
[software UI](../systems/kestrel-software-ui.md),
[browser interfaces](../systems/ascent-browser-interfaces.md) and
[P1 validation](../experiments/p1-source-stack.md) for board-specific results.

The [P1 source publication checkpoint](../firmware/p1-boot-and-recovery.md#source-publication-checkpoint)
records the paired platform/application draft submissions and unchanged tested
commits after branch renaming. Draft submission is distinct from upstream merge
and from a released package with its final dependency pin.

## Hardware, vendor images and analysis tools

The [AR8030 analysis method](ar8030-analysis-method.md) records input identities,
synthetic ELF/ABI choices, the pinned REA/Ghidra workflow, custom-instruction
limitations and what retained evidence checks actually verify.

The flash-marking study used the
[GD5F1GM7RE family page](https://www.gigadevice.com/product/flash/spi-nand-flash/gd5f1gm7re),
[GigaDevice selector](https://www.gigadevice.com/product/flash/spi-nand-flash) and
[Linux GigaDevice driver](https://github.com/torvalds/linux/blob/master/drivers/mtd/nand/spi/gigadevice.c).
RE-family definitions do not resolve the reported RF package suffix. The
[device study](../devices/ascent-vrx.md#flash-identification) separates marking,
family reference and later driver-log evidence.

Official package discovery used the
[Caddx download center](https://caddxfpv.com/pages/download-center),
[Ascent downloads](https://wiki.caddxfpv.com/en/Download/Ascent-HD-Firmware) and
[release history](https://wiki.caddxfpv.com/en/Ascent-HD-Series/firmware-release-note).
Each firmware article retains exact version/hash and extraction/execution scope.

Offline tooling included Capstone/ELF inspection, `dnfile`/`dncil` for .NET CIL,
SquashFS readers and independent `unsquashfs`, and public-key parsing for image
verification. Successful parsing does not prove acceptance of a modified image.
Bounded credential searches did not recover the original root password; later
access used a separate startup path.

[CODA bits](https://github.com/pH5/coda-bits/tree/61c8cc4a3e398c3170465e140922283c7628296d)
provided an explicitly incomplete older BIT-processor comparison. Its code
templates, word-address units and control-flow encodings do not establish a
Wave521C ISA. The related
[StarFive SDK snapshot](https://github.com/starfive-tech/soft_3rdpart/tree/b60da16be1b36453aa46599889c28310fd148fb8)
helped separate CODA980 decoder low-delay commands from encoder SFS interfaces;
the [matched Artosyn binary audit](../experiments/ar9311-slice-decoding.md#why-the-exported-low-delay-api-does-not-qualify-wave521c)
supplies the product/backend gate rather than assuming SDK equivalence.

The early Python/Rust Artosyn-tool searches did not find the matching serial
frame. Some ground-station package paths returned 404; `ar8030-rs` and
`zig-ar803x` were metadata-only leads. The older Artosyn SDK repository supplied
no usable AR9311 SD-boot procedure. These limited results are not evidence that
the missing capabilities cannot exist elsewhere.
