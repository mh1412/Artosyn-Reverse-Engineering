# Open receiver stacks and the AR9311 port

[Systems](README.md) · [AR9311](../soc/ar9311.md)

The October 4–5, 2026 assessment compared three different software boundaries:
the stock Gnd485 receiver, MissingLynk's AR9311 reference platform, and the then
RK3568-based fpvOS receiver. Later work added an AR9311 fpvOS port and a standalone
stock-kernel Kestrel UI. The early statement that fpvOS only targets RK3568 is
therefore historical, not the scope of the later research.

## Layers in the original comparison

| Layer | Stock Gnd485 | MissingLynk reference stack inspected in October | Original fpvOS VRX Pro target |
| --- | --- | --- | --- |
| Application processor | AR9311, two Cortex-A53 cores | AR9311 / Proxima-9311 | RK3568 / Mali |
| Kernel | Vendor Linux 4.9.38 | Linux 6.18.36 plus platform drivers/patches | Extracted vendor Rockchip kernel |
| Userspace | BusyBox/glibc, vendor media/radio applications | Alpine 3.24.1/musl/OpenRC, project media/radio applications | Buildroot, Kestrel receiver |
| Radio host interface | Vendor SDIO daemon and device | Open SDIO transport and `ml-linkd` | Vendor USB daemon/client library |
| Video/display interface | Artosyn MPI/HAL, CUSE/Binder/MMZ | V4L2 wave5, DRM/KMS, GStreamer | Rockchip MPP, DRM/GBM |
| UI | Qt linuxfb | LVGL overlay | GLES overlay |
| Boot/storage reference | NAND, shared kernel/DTB and paired application/data banks | Reference-board boot/container and NAND profiles | Vendor boot inputs adapted for SD boot |

These version numbers describe inspected snapshots. They are not a list of
current upstream releases. The original comparison's modified stock receiver
used a 17.5.3 root-access wrapper with a selected 17.5.8 startup continuation;
that combination was not an untouched vendor installation.

MissingLynk separates wrapper/profile tooling, kernel, userspace, rootfs and
unofficial hardware documentation. Its reported reference-board results included
SMP/clocks, display overlays, H.264/H.265 codec operation, SDIO association and
video, SD storage, GPIO, RTC and GStreamer DVR. JPEG, some scaler/audio functions
and parts of camera/ISP coverage remained gaps in the inspected assessment.
Those reports concern VR04-related devices, not measurements on Gnd485 or proof
of P1 camera compatibility.

## What remains vendor-derived

Replacing Linux userspace does not replace every firmware image running inside
the system. The inspected MissingLynk stack still required radio firmware and
configuration, Wave521C microcode, and applicable camera tuning/generated tables.
Its normal runtime did not launch the vendor Linux media service, but vendor
binaries and dumps remained reverse-engineering inputs.

Original VRX Pro fpvOS reused boot/kernel inputs and baseband, Mali and RGA
runtime components. Rockchip MPP was built from open source; its extracted vendor
copy was a fallback, skipped in the normal installation. Rockchip MPP and
Artosyn MPI/HAL are distinct implementations despite the similar names.

Later [Open8030](../firmware/open8030-runtime.md) work changes the radio-firmware boundary
for its validated matched AP/DEV configurations. It does not retroactively
make every earlier stock-radio experiment independent of vendor firmware.
The reviewed SDK assessment did not establish a usable open NPU/DSP toolchain.

## Board porting requirements

A shared SoC provides driver reuse opportunities; it does not identify the
display bridge, GPIO wiring, flash layout, reserved memory or product protocol.
The early MissingLynk profiles were `betafpv-vr04-goggle` and
`betafpv-vr04-air`. Gnd485 needed its own description and validation.

In particular, the Gnd485 map has a shared kernel and DTB with paired application
and data banks. Reference scripts expecting `kernel1`/`dtb1` cannot be adopted
solely because two application slots exist. The receiver also outputs HDMI
through its board-specific display chain rather than the reference goggle panel.

| Original open question | Later evidence in this wiki | Remaining boundary |
| --- | --- | --- |
| Recoverable custom boot | [Gnd485 USB RAM boot](../firmware/gnd485-ram-boot.md) | Does not depend on declaring the early unreadable UART solved |
| Memory, NAND and bank selection | [NAND/boot layout](../firmware/gnd485-nand-and-gpt.md), [fpvOS update contract](../firmware/gnd485-fpvos-update.md) | Shared boot resources and failure cases still need their own treatment |
| GPIO, storage clocks and watchdog | [Platform validation](../experiments/ar9311-platform-validation.md) | Not a complete peripheral/power-management qualification |
| Hardware decode and HDMI | [Display pipeline](ar9311-display-pipeline.md), [HDMI experiments](../experiments/ar9311-hdmi.md) | Fixed-mode successes coexist with unresolved visible-artifact investigations |
| SDIO radio | [Gnd485 radio bring-up](../experiments/gnd485-sdio-radio.md) | Named firmware/profile and bench conditions apply |
| Lightweight stock-kernel UI | [Kestrel software UI](kestrel-software-ui.md) | Physical radio/media still delegated to vendor services |
| Air-side source camera and recovery | [P1 source stack](../experiments/p1-source-stack.md) | SC231AI P1 is distinct from the NT99235 reference device |

## UI portability is a separate problem

The initial Kestrel assessment used commit
`e60d59e4618448040ea35e54b0e2cd4244a4157e`. Decoded video and OSD already had
separate threads/planes, and themes and planar canopy geometry were reusable.
However, OSD state, shader/texture ownership and presentation were intertwined;
the build required EGL/GLES/GBM even when runtime OSD was disabled.

The suggested `UI_BACKEND=gles|lvgl|both` selector was a proposal, not an existing
build option. Later standalone work actually used `KESTREL_UI_ONLY`, optional
service-only builds, and an LVGL renderer contract. It reproduced perspective
text on the CPU while retaining hardware video underneath. It did not add a
supported GLES driver to AR9311.

GE2D and the display compositor are fixed-function blocks. Their existence
does not establish shader support or make the RK3568 renderer portable. The
[GE2D investigation](../soc/ar9311-ge2d.md) instead established a specific
bitmap/alpha ABI and bounded offscreen hardware results.
