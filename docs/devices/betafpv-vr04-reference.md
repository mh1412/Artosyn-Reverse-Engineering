# BETAFPV VR04 reference goggle and air-unit boundaries

[Devices](README.md) · [Open receiver stacks](../systems/open-receiver-stacks.md)

## Identity and scope

The inspected MissingLynk reference work targets `betafpv-vr04-goggle` and
`betafpv-vr04-air`, using AR9311/Proxima-9311 and an AR8030 companion radio.
The kernel snapshot is `e3885c6394a7025eacd3a23f473276d9823beb25`; userspace is
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`. Findings below consolidate those
retained sources and their reported hardware results, not new measurements.

The reference air camera is **NT99235**. Its `P1_SKY` firmware labels must not
silently equate it with the later [SC231AI P1 Air](betafpv-p1-air.md). Goggle
panel, keypad, power rails, radio reset and memory reservations likewise do not
describe [Gnd485](ascent-vrx.md).

Related studies cover [stock access and package boundaries](../firmware/vr04-stock-access.md),
[link/binding policy](../systems/vr04-link-control.md),
[media-session framing](../protocols/vr04-video-session.md), and
[stock UI/channel controls](../protocols/vr04-stock-ui.md).
The [stock media graph](../systems/vr04-stock-media.md),
[air encoder benchmark](../experiments/vr04-air-encoder-benchmark.md) and
[receiver latency study](../experiments/vr04-receiver-latency.md) distinguish
recovered interfaces, synthetic workloads and receiver-local measurements.
The [open media/HUD study](../systems/vr04-open-media.md) covers the later source
pipeline, including recording overlays and playback. Its
[FC canvas](../protocols/vr04-msp-canvas.md) and [local IPC](../protocols/mlm-ipc.md)
have separate transport and validation boundaries.

## Driver and firmware boundaries

| Function | Reference implementation |
| --- | --- |
| NAND | Custom AR9301 spi-mem controller, PIO/FIFO; mainline SPI-NAND/MTD/UBI above it |
| GPIO | Custom Artosyn gpiochip provider; mainline consumers where applicable |
| Buttons, temperature, PWM | Custom providers with IIO, adc-keys and PWM/backlight interfaces |
| Display | Custom Artosyn VO, DSI glue and QY45043A0 panel; DRM/KMS and DesignWare DSI core |
| SD and radio host | DesignWare MMC core with Artosyn clock glue; separate AR8030 SDIO driver |
| Video | Patched V4L2 Wave5 driver; proprietary codec microcode |
| DMA, USB, SPI, CPU PMU | Mainline IP drivers with board/device-tree configuration |
| RTC | External DS1307 at I²C0 address `0x68` |

The reference runtime retired its vendor-MPP kernel compatibility modules in the
recorded July 12, 2026 pivot. Those sources remained compile-checked references;
their presence is not evidence that they were shipped or loaded. DRM overlay and
V4L2/dma-buf paths replace the old CUSE framebuffer and MPP memory plumbing.
Radio blobs/configuration, codec microcode and applicable camera tuning/tables
remain separate vendor-derived inputs.

Reported goggle results include both A53 cores at 800 MHz, all eight vendor CPU
OPPs checked with PMU counters, SD storage, radio video, exact decoded pictures
and a two-decoder plus 1080p60 DVR configuration. The reference codec image is
999,616 bytes, revision 329715; it is not the Ascent decoder image described in
the [firmware reconstruction](../firmware/wave521c-reconstruction.md).

The [NT99235 camera study](../soc/ar9311-nt99235-camera.md) reports a working
1080p60 capture chain within successful sessions and a provenance model of 1,260
distinct ISP offsets: 1,258 classified as regenerable and two hardware-owned
values needing further device evidence. This classification is not an exhaustive
executed-write audit. The later driver keeps the sensor pipeline running between captures
to avoid the historical full teardown/restart failure. It also retains incomplete
adaptive-stage hardware checks, inherited camera clocks and unvalidated VIF bypass.
The [ISP study](../soc/ar9311-nt99235-isp.md) and
[capture validation](../experiments/nt99235-camera-validation.md) separate those
limits from operating-point register agreement and image corrections.
The [media validation record](../experiments/vr04-media-validation.md) distinguishes
historical encoder-instance failures from later recovery, live-control and DMA patches.

## Boot and memory profile

The reference boot layout has a **6 MiB `kernel1` partition** and separate
`dtb1`. Its codec pool is physical `0x29200000`, size `0x06e00000` (110 MiB),
within the board's 256 MiB RAM range `0x20000000–0x2fffffff`. These addresses
and paired boot partitions differ from Gnd485.

The current tree also reserves 1 MiB for ramoops at `0x29100000`, immediately
below the media pool. This leaves a 145 MiB lower region before other kernel
reservations, not 146 MiB of usable Linux memory. Codec and userspace MMZ heap
reference the same 110 MiB pool. The complete
[device-tree study](../firmware/vr04-device-tree.md) records the 20-partition map,
retained legacy nodes, display graph and limits of static resource declarations.

The recorded boot container is OTRA plus legacy uImage carrying LZ4. Raw ARM64
`Image` failed with a magic error; LZ4 linked blocks failed with `-93`.
Independent-block framing used `FLG=0x64`, `BD=0x70`, with container load at
`0x24000000` and decompression to `0x200a0000`. RAM boot required explicit
`mtdparts`, which SPL supplied during flash boot. The secondary CPU used the
board's `ar-spin-table` protocol, rather than Gnd485's PSCI path.

The build pins kernel 6.18.36, GCC 14.2.0 and deterministic metadata. Its verify
mode compares two builds; fast incremental mode is explicitly outside that
reproducibility claim. The container base still needed a digest pin for full
base-image reproducibility. Kernel fit examples around 96.5–96.9% are snapshots,
not guaranteed space in a later build. Boot-critical drivers are built in;
large off-boot components such as DRM and codec are modules.

The retained status distinguishes a working open slot-B rootfs from the **current
candidate kernel**, which was RAM-boot proven but still pending flash. RAM boot
changes the kernel, not the flashed userspace. The recorded reference flasher
wrote inactive B while running A, then verified the actual flashed bytes before
a separate slot-selection step. This is reference history, not a Gnd485 flashing
procedure; its [shared kernel/DTB layout](../firmware/gnd485-nand-and-gpt.md) needs
its own update contract.

## Goggle display and shared rail

The picture path is VO at `0x08810000` → DesignWare DSI at `0x08850000` →
QY45043A0 1920×1080 panel. VO uses GIC SPI 70, architectural interrupt ID 102;
Linux IRQ numbering is not a fixed physical identity. DSI `GEN_HDR`/`GEN_PLD_DATA`
offsets `0x6c`/`0x70` corroborate the DesignWare interface.

The vendor panel initialization has 519 four-byte DCS tuples. Its tail sends
Sleep-Out with 32 ms delay and Display-On with 40 ms delay; register `0x98`
returns panel ID `0x96`. A configuration value 1488 was interpreted as a
possible DSI bitrate, not independently established. Other named panel/bridge
variants in the binary do not identify the fitted panel.

| Global GPIO | Reference goggle role |
| --- | --- |
| 23 | AR8030 active-low reset |
| 42 | Horizontal/source-scan strap |
| 43 | Panel reset/enable |
| 95 | Second panel rail/enable |
| 100 | Shared panel-power / SD I/O 3.3 V rail |
| 107 | Vertical/gate-scan strap |

Panel bring-up drives 95 and 43 low, waits 10 ms, raises 43, waits 10 ms, raises
95 and waits another 10 ms. Orientation straps 42 and 107 must already be low
when GPIO100 rises. Leaving them floating produced the observed 180° rotation;
the correction was power-on strap ownership, not a display rotation command.

The open path provides DRM primary/ARGB4444 overlay planes and fbdev emulation
for the console. It does not provide a U-Boot framebuffer or a pre-kernel splash;
the panel starts after the required drivers probe. Frame-done interrupt bit 0
at VO interrupt-enable offset `0x1480` supplies vblank/page-flip completion,
replacing the old 100 ms MPP polling path. Dynamic pixel-clock changes have
separate [DSI tolerance limits](../soc/ar9311-platform.md#reference-platform-clock-work).

## Storage and clocks

The removable card host is `0x01c00000`, GIC SPI 49. Its SD I/O supply shares
GPIO100 with the panel, so the reference DT holds it with one GPIO hog rather
than allowing MMC to power-cycle a separately claimed regulator. Card-core
power routing remains unresolved. Native CDETECT replaced the early polling
overlay; device names such as `mmcblk1` vary with probe order.

Recorded operation is 50 MHz, four-bit, 3.3 V SD High-Speed, about 21–22 MB/s
direct I/O. A 16 MiB write/read took 781/762 ms; a 64 MiB read reached 22.4 MB/s.
The exFAT workload included paced and full-speed 1 GiB writes with block MD5
verification and clean unmount. Fixed 3.3 V I/O prevents the required 1.8 V UHS
switch; this is a board limitation. The report did not resolve IDMAC versus PIO.

The inspected MMC glue defaults to SEL `0x80`, phase 0. An older board table's
`0x87`/2 pair is superseded for this reference snapshot. Its implementation
writes CFG's low five bits, SEL and a delay; a `bus_hz` override changes the
software frequency, **not** an automatically selected hardware tap. Named gate
registers are mapped but not written. Old bits-22/23 comments do not establish
clock ownership; later [Gnd485 gate recovery](../soc/ar9311-platform.md#mmc-source-clocks)
provides separate evidence for its provider.

The failed SD experiment force-wrote the shared `0x0a108000` control word as
`0x81`, after which CMD8 responses failed. That does not prohibit the scoped
ADC/protemp enable-bit updates described in the [analog study](../soc/ar9311-analog-and-pwm.md).
The MMC glue leaves those shared control registers alone.

## Status LEDs and other board services

The goggle has **three** addressable RGB pixels on SPI `0x01102000`, using
6.25 MHz, eight-bit transfers and GRB order. One LED data bit expands to one SPI
byte, giving 72 data bytes for the chain. Stock uses `c0` for zero and `fc` for
one, and shows orange `(G=40,R=255,B=0)`: blinking when searching, solid when linked.

The open implementation changed zero to **`80`** after all-red frames produced
red/pink/orange across the chain with `c0`. At 6.25 MHz this shortens zero's high
pulse from 0.32 to 0.16 µs; one remains `fc`. It adds 320 low bytes, about 410 µs,
for latching. The separate `ml-ledd` service animates at about 33 Hz and accepts
nonblocking Unix datagrams on `led.sock`: last command wins, and producers
periodically reassert state. The retained link policy is breathing red without
a usable link, breathing orange at standby 15 fps and solid green at 60 fps.
These are open-runtime policies, not the stock orange-only indication.

The reference `P1_SKY` air firmware instead controls plain GPIO LEDs: line 0 red,
line 1 green, both active low, with observed current competition consistent with
the reported shared series supply. High-impedance lines glow faintly. Stock
initialization drives both low; binding alternates `{0,1}` and `{1,1}` at a
40 ms half-period, making red visible while green is off. The open `leds-gpio`
defaults red off/green on, and needs no WS2812 daemon. This mapping remains tied
to that studied air board; it is not inferred from the goggle SPI interface.

The goggle buzzer, brightness ladder, keypad and calibrated battery/temperature
interfaces are detailed in [analog and PWM peripherals](../soc/ar9311-analog-and-pwm.md).
The external DS1307 kept time while powered; its removed backup battery explained
loss across power-off. This differs from the unresolved Gnd485 internal RTC.
JPEG, scaler and audio had separate gaps; stock audio was disabled in this
reference configuration. USB gadget composition was limited to three bulk-IN
FIFOs, and active SD recording was kept separate from block-exclusive mass storage.

See [reference SDIO transport](../protocols/vr04-sdio-transport.md) for boot IDs,
IP framing and the distinction between association and useful video delivery.
