# Reference P1_SKY stock access and package boundaries

[Firmware](README.md) · [Reference board](../devices/betafpv-vr04-reference.md)

## Identity and access observations

The historical reference air/goggle investigation examined `P1_SKY`/`P1_GND`
firmware `1.0.44.rel`, build `09de6e7`, with a live root shell. These product
strings do not resolve the NT99235-versus-later-SC231AI board distinction;
the [device boundary](../devices/betafpv-vr04-reference.md#identity-and-scope)
must accompany them. This article reports the retained investigation, not a new
login, package extraction or firmware installation.

Stock air and ground both exposed USB Ethernet at `192.168.3.100`, hostname
`art_sirius`, with the recorded root password `artosyn`. Hostname alone could
not distinguish them. The air identified itself through `product_version=P1_SKY`,
`ar_lowdelay -m 2 -t 1` and `auto_sync -sky`. Both devices using that stock address created a collision when connected together.

Air-side SSH was also reachable at `10.0.0.100:22` over an associated RF link.
An earlier “ping only, no shell” conclusion was incorrect: the goggle lacked
an originating SSH/netcat client, and its scripted port-8822 DNAT rule could
not run because `iptables` was absent. A userspace TCP relay from goggle port
8822 to the air SSH port established the recorded shell from stock slot A.

The live air showed listeners on 22 and 50000, no demonstrated Wi-Fi access
point, and no RTSP listener on 554. Presence of `libar_minirtsp.so`, a 143,768-byte
library with HEVC/RTP/RTSP routines, established an available code path, not a
running camera RTSP service.

## Filesystem and persistence

The examined runtime used BusyBox 1.25.0, Dropbear and an ARM64 Linux 4.9.38
platform reporting Proxima-9311. Its read-only root was SquashFS on
`ubiblock0_0`, about 15.5 MiB stored and 34.5 MiB unpacked, with 629 inodes.
Writable `/usrdata` was UBIFS. The startup script invoked
`/usrdata/run_dbg.sh` when present, providing the recorded persistent-hook
surface. This board-specific hook does not substitute for the different
[Ascent access mechanism](ascent-root-access.md).

The factory filesystem held per-unit certification checked by stock media
startup. Its contents remain part of the stock recovery boundary. The record also found
recording/file-transfer/update executables and IMU-log support. Symbol presence
does not qualify every recorder, audio or IMU function on the physical board.

Generic bundled device-tree entries described NOR/NAND controllers and an early
Sirius demo board. They are not a physical inventory of fitted flash chips.
## OTA container versus mounted rootfs

The retained study identifies the supplied images as OTRA containers carrying
DTB, LZ4 kernel and LZ4 SquashFS content. A rootfs magic at image offset
`0x4b4233` belongs to the examined package, not a generic search constant.

The OTA rootfs did not extract directly with ordinary SquashFS tools even though
its 32-bit superblock fields were plausible. Its 64-bit pointer region
`0x20–0x5f` differed from a normal mounted-rootfs superblock. The live ground
filesystem was already readable/deobfuscated, supporting an install-time
transformation rather than encrypted filesystem reads. Reusing a ground-derived
XOR mask did not decode the air package.

The record proposed a custom pointer pack/unpack and identified
`artosyn_upgrade` as the installer to inspect. It did not supply a complete
verified transform here. A live filesystem dump can provide files without
solving the OTA transform; that does not prove a modified OTA image will install.
Stock `unsquashfs` 4.5.1 read the deobfuscated dump in that investigation, with
vendor-variant tooling considered a fallback.

Two 65-byte P-256 public points were identified among installer constants.
Other long strings matched standard curve parameters and crypto test vectors,
not recovered device secrets. No matching private signing key or secret symmetric
key was established by that inspection. Public verification material is not
authority to sign a replacement image, and a search without a recovered key
does not prove the absence of encryption in every other firmware component.
