# Ascent air/ground architecture and radio-role reuse

[Systems](README.md) · [Gnd485](../devices/ascent-vrx.md) ·
[Sky482](../devices/ascent-lite-vtx.md)

The matched 17.5.8 Gnd485/H Sky comparison found a common Linux application/radio
organization with different host ABIs, media drivers and boot/update machinery.
Shared radio interfaces make source-level reuse plausible; they do not make
receiver binaries or firmware packages interchangeable with the transmitter.
The comparison was offline on October 7, 2026. Later live Sky482 findings are
identified separately below.

## Application hosts

| Boundary | Gnd485 receiver | Sky482/Lite transmitter |
| --- | --- | --- |
| Host platform | AR9311/Proxima, live dual Cortex-A53 | Hi3516CV610 SDK profile, ARMv7 reference binaries |
| Kernel | 4.9.38 AArch64 | 5.10.0 module target in 17.5.8; later 18.21.10 live kernel 5.10.221 |
| ABI / loader | ELF64, glibc, `/lib/ld-linux-aarch64.so.1` | ELF32 ARM EABI5, soft-float flag, musl, `/lib/ld-musl-arm.so.1` |
| Compiler evidence | Linaro GCC 7.5.0 | GCC 10.3.0; SDK strings identify musl 1.2.3 |
| Reference BusyBox | 1.25.0 | 1.34.1 |
| Root format | SquashFS in UBI | UBIFS in UBI |
| Main application | `ar_ldy_gnd` | `ar_ldyhs_sky` |
| Radio daemon | `daemon_sdio` | `daemon_sdiov12` |
| Manufacturing service | `ar_fpv_mp_service` | `ar_fpvhs_mp_service` |
| Media interface | `AR_MPI_*`, dynamic MPI/HAL libraries | `ot_mpi_*` / `ss_mpi_*`, substantial code embedded in backend |
| Primary video function | Receive, decode, display | Camera/ISP/process, encode, transmit |
| UI | Qt `GlassesUI`, linuxfb | No corresponding separate Qt UI identified in selected startup/tree |
| Normal USB | RNDIS + ACM, `1d75:0101`, `CX11` | ACM, `1d76:0101`, `CX20`; alternative RNDIS branch |
| Updater | `ar_fpv_upgrade` | `ar_fpvhs_upgrade` |

SMP module metadata establishes a build option, not the physical core count.
The VTX SDK profile does not substitute for a silicon marking/ID. Likewise,
the receiver's `mmz_allocator=hisi` label does not override its live Proxima DT
identity. Shared “MPP/MPI” terminology does not establish a common binary API.

The receiver live map describes 512 MiB, split into 256 MiB Linux at `0x20000000`
and 256 MiB MMZ at `0x30000000`, with 16 MiB CMA inside Linux memory. The selected
Sky482 reference loader combines `mem=64m` with 64 MiB MMZ at `0x44000000`,
consistent with a configured 128 MiB map from `0x40000000`. Generic-loader
comments describing 32+96 MiB do not describe that selected configuration.
Later live Sky482 command-line evidence confirms `mem=64m`; neither observation
is a destructive physical RAM-capacity test.

The receiver backend imports 143 selected `AR_MPI_*` names; the VTX defines
1,466 selected OT/SS MPI names and lists only libc/libgcc as ELF dependencies.
These selected symbol counts support different media implementations, not a
percentage of shared source. Sky482 startup selects `cv2004`, `fpv405_qfn` and
an encoder-channel maximum of two. Later boot logs corroborate those software
selections, without independently identifying the fitted sensor package.

## Shared radio boundary, different host ABI

Both native SDIO modules advertise `4152:8030` and `4152:8031`, accept firmware/
configuration names, and expose `/dev/artosyn_sdio`. Both backends share 21
selected public `bb_*` names covering host/device lifecycle, enumeration, IOCTL
and socket operations, plus five helpers. The updaters share 88 selected
AR_FPV/timer names. These are interface-lineage observations.

Static `bb_host_connect` paths use local TCP creation and vendor frame packing/
unpacking. In the examined `bb_ioctl` handles, the mutex lies at offset `0x90`
on AArch64 and `0x60` on ARM32. Identical function names therefore do not make
raw structures, libraries or preload shims interchangeable. Local TCP transport
also does not mean the RF packets are ordinary IP traffic.

The transmitter must retain its own driver, daemon and media BSP when testing
radio-role changes. The [AR8030 loading study](../firmware/ar8030-loading-and-driver.md)
describes the separate coprocessor's container and transport.

## Matched SKY/GND radio images

These images contain RISC-V radio code, distinct from the ARM Linux applications.

| Field | SKY | GND |
| --- | ---: | ---: |
| Complete bytes | 423,424 | 424,960 |
| Header | 64 bytes, magic `0x41528030`, type 1 | Same |
| Bootstrap at file offset 512 | 944 bytes | Identical 944 bytes |
| Bootstrap destination | `0x0a098a00` | Same |
| Embedded config at offset 1,536 | 16,384 bytes | Different bytes, same size |
| Config destination | `0x00200000` | Same |
| Runtime at offset 17,920 | 404,992 bytes | 406,720 bytes |
| Runtime destination | `0x00204000` | Same |

Both startup prefixes decode coherently as RV32+C, with machine-mode CSR
initialization and memory clearing. Shared strings identify software 0.1.12 and
SDK 1.2.01-01, with builds at 19:03:14 and 19:04:43 on April 17, 2026.
Different payload hashes establish distinct builds, not their semantic distance
or an AP-only/DEV-only implementation.

| Artifact | SHA256 |
| --- | --- |
| Shared bootstrap | `a9e59b0e05c42042e27c35ca998c83326aa9d8ab0cea294bc2d0f7199929df68` |
| SKY full image | `e66dae0ab8d9ecb248239af45f298dc334ad602aa6f5f4700a92b299a51bb51b` |
| GND full image | `dbdea6c72bb0d5f75ffa395693720967b19c8993f0ad6419083f49a3c1ac3705` |
| SKY main payload | `bd452d76f15c9fb12ef506acd3912b83970068ba67695bc3aa135c4d7ff76135` |
| GND main payload | `9f4b661bf8b8cecc4dd989d0c9b7d22f32253512784e06f2f3f51816806c5b49` |

The header region named `signature` in the recovered layout contains JSON in
these images, not an identified RSA signature. Both embedded configurations
default to **AP**, with both AP/DEV clocks at 200 MHz. Separately supplied
startup configurations differ:

| External configuration | Role | AP clock | DEV clock |
| --- | --- | --- | --- |
| SKY | `ap` | 200 MHz | 100 MHz |
| GND | `dev` | 100 MHz | 200 MHz |

The native VTX loader sends external configuration after embedded configuration,
to the same address. Thus filenames alone do not select role. Its inspected
probe enters download for ID `0x8030`, sends the image header to `0x002f0040`,
then bootstrap/config/runtime and the overriding JSON before a zero-length
completion packet. It does not guarantee another download merely because a
driver is inserted against an already-running chip.

Matching bootstrap, destinations and upload mechanism support a possible GND
runtime experiment on VTX hardware. They do not prove RF-board compatibility.
Later reverse engineering found different hard-coded startup selector sequences
reaching register helpers near `0x0a10a000` and `0x0a090140`; external role JSON
alone cannot establish equivalent board initialization. The generic decoder's
opaque `custom0` instruction and selector-to-pin mapping remained unresolved
in that investigation. Preserve board calibration, GPIO and PA/LNA distinctions.

## What a headless receiver conversion would require

The proposed application forwards compressed video to a computer:

```text
stock air unit → RF → VTX-board radio in a validated ground role
                    → native SDIO driver/daemon
                    → headless ground controller
                    → HEVC deframer/RTP → USB Ethernet → host decoder
```

This avoids needing an onboard decoder, HDMI or Qt. It still needs a ground
controller to establish the peer, lifecycle, video socket and control exchange.
The stock ground path opens logical video port 3 (audio separately on port 2)
and calls `bb_socket_read` before video parsing. The stock Sky path frames
outgoing data and calls `bb_socket_write`, including partial-write handling.
The [deployed VRX proxy](ascent-groundstation.md) copies an existing session;
it does not perform that ground initialization or pairing.

The October 7 proposal had no candidate-board access or conversion result.
Later October 16 work established Sky482 identity, root shell and USB Ethernet
after repair, resolving some prerequisites. It did **not** establish stock-GND
radio reception on the repurposed transmitter. Later
[Open8030 matched AP/DEV results](../experiments/open8030-validation.md) concern
their named replacement firmware/board pairs and must not be substituted for
this distinct stock-peer conversion claim.

| Required proof | What would establish it |
| --- | --- |
| USB/software forwarding | Target-built ARMv7/musl replay, host decode, measured throughput and bounded memory |
| Radio runtime | Clean download boundary, running queries, effective role and retained board calibration |
| Stock-link reception | Pairing/control plus actual parameter sets and independently decoded stock video |
| Sustained system behavior | RF/peer/host reconnects, late join, bitrate changes, thermal and resource measurements |
| Persistent deployment | Reboot and rollback evidence after runtime qualification |

The optional RNDIS startup branch consumes `/factory/run_rndis_flag`; it is
alternative Ethernet mode, not proof of a permanent combined ACM/RNDIS gadget.
The configured device address is `192.168.3.104`, host/gateway `.204`. USB 2.0
descriptors and the VRX's approximately 12.7 Mbit/s example stream do not measure
VTX forwarding capacity.

An examined public AR8030 transport example supplies a source-level DEV socket
receiver and RTP output but introduces its own 22-byte TX/RX chunk format.
That is a building block, not evidence of stock Ascent payload compatibility.
The outer daemon frames and inner video bytes from `bb_socket_read` likewise
require different parser entry points.

Media adapters, native ABI rebuilding, board initialization and boot selection
remain target-specific. The [air-unit updater](../firmware/ascent-air-unit-updater.md)
also differs from the receiver's RSA/GPT path. A receiver host firmware image
is not a shortcut to this conversion.
