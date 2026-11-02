# VR04 goggle device-tree contract

[Firmware](README.md) · [Reference board](../devices/betafpv-vr04-reference.md) ·
[Platform peripherals](../soc/ar9311-platform.md)

This is a complete source review of the goggle's `proxima-9311.dts` at
MissingLynk kernel revision `e3885c6394a7025eacd3a23f473276d9823beb25`.
It describes the declared configuration and its ownership boundaries. Compiling
this tree does not prove that every node probed, every module was deployed or
every claimed rate was measured. The air board and Gnd485 have separate layouts.

## Boot handoff and CPU policy

The tree declares two Cortex-A53 CPUs using the custom `ar-spin-table` release
method. CPU1's release address is `0x00100000`; ordinary `spin-table` is not the
recorded SPL handshake. Both CPUs reference the same clock and shared OPP table:
24, 100, 200, 333.333333, 400, 600, 666.666666 and 800 MHz. All eight entries
specify 900,000 µV and 150,000 ns transition latency; 24 MHz is the suspend OPP.
These entries describe policy, not eight independently verified voltage settings.

`serial0` selects the DesignWare UART at `0x01500000`, with register shift 2 and
32-bit accesses. The chosen console is 1,152,000 baud, 8N1. Bare `earlycon`
uses that node; replacing it with a generic explicit MMIO argument can lose the
register shift. The node supplies a 150 MHz clock to the full driver but omits
a UART `clock-frequency` property so earlycon retains the bootloader setup.

The chosen command line specifies `ubi.mtd=18`, `root=ubi:rootfs`, UBIFS and
read/write mounting, plus `keep_bootcon` and `ignore_loglevel`. The retained SPL
rewrites the numeric UBI field for the active slot. A stale partition comment
suggesting a named `ubi.mtd=userapp1` conflicts with this boot contract: the
recorded rewrite mangled a name into `18erapp1`. Index 18 depends on the stated
partition/master enumeration and is not a cross-board constant.

## DRAM, media allocation and persistent logs

Ranges below use an **exclusive** end address.

| Declared range | Size | Purpose |
| --- | ---: | --- |
| `0x20000000–0x30000000` | 256 MiB | Entire declared DRAM bank |
| `0x20000000–0x29100000` | 145 MiB | Lower region before other kernel reservations and allocations |
| `0x29100000–0x29200000` | 1 MiB | `ramoops`, `no-map` |
| `0x29200000–0x30000000` | 110 MiB | Shared media pool, `no-map` |

The two top reservations are adjacent and exactly reach the end of DRAM.
Thus the older comment saying ordinary RAM ends at `0x29200000` omits the later
persistent-log reservation. The 145 MiB figure is an address-space calculation,
not available-memory or `MemTotal` output.

The Wave5 codec and `missinglynk,mmz-heap` reference the **same** reserved-memory
node. They share allocator capacity in the matched implementation; a 110 MiB
codec pool and a separate 110 MiB userspace pool must not be budgeted. This tree
does not declare a reusable CMA region. Historical comments about 64 MiB CMA
describe a different allocation mechanism and do not create it here.

The ramoops layout requests 256 KiB console, 128 KiB ftrace, 128 KiB pmsg and
128 KiB records in the remaining 512 KiB: four nominal record slots. `flags=1`
requests per-CPU ftrace storage, and `max-reason=5` sets the dump-reason limit.
Zone sizes include implementation metadata; they are not guaranteed application
payload capacity. `no-map` excludes these regions from the ordinary linear map.
Retention still requires the reset/boot path to preserve the DRAM contents;
the tree does not establish power-loss persistence or guaranteed crash capture.
The [breadcrumb review](../systems/vr04-display-diagnostics.md#persistent-breadcrumbs)
documents userspace write and interpretation limits.

## Declared SPI-NAND partition map

All offsets are raw flash offsets in this tree, not OTRA package offsets or
UBI logical-volume addresses. The 20 entries are contiguous through exclusive
offset `0x07d00000` (125 MiB). That endpoint does not establish the fitted chip's
capacity, bad-block allowance or an updater's write policy. None of these
partition nodes declares `read-only`.

| Label | Offset | Size |
| --- | --- | --- |
| `spl0` | `0x00000000` | 256 KiB |
| `spl1` | `0x00040000` | 256 KiB |
| `spl2` | `0x00080000` | 256 KiB |
| `spl3` | `0x000c0000` | 256 KiB |
| `gpt0` | `0x00100000` | 256 KiB |
| `gpt1` | `0x00140000` | 256 KiB |
| `vendor` | `0x00180000` | 512 KiB |
| `factory` | `0x00200000` | 6 MiB |
| `env0` | `0x00800000` | 384 KiB |
| `env1` | `0x00860000` | 384 KiB |
| `uboot0` | `0x008c0000` | 768 KiB |
| `uboot1` | `0x00980000` | 768 KiB |
| `kernel0` | `0x00a40000` | 6 MiB |
| `kernel1` | `0x01040000` | 6 MiB |
| `dtb0` | `0x01640000` | 384 KiB |
| `dtb1` | `0x016a0000` | 384 KiB |
| `userapp0` | `0x01700000` | 45 MiB |
| `userapp1` | `0x04400000` | 45 MiB |
| `usr_data` | `0x07100000` | 6 MiB |
| `usr_log` | `0x07700000` | 6 MiB |

## Driver ownership and resource declarations

GIC SPI numbers below are binding values. Their architectural interrupt IDs
are SPI + 32; Linux's allocated IRQ numbers need not equal either.

| Block | Base | GIC SPI |
| --- | --- | --- |
| UART | `0x01500000` | 11 |
| I²C0 / external RTC bus | `0x01200000` | 32 |
| Active watchdog | `0x01600000` | 37 |
| Status-LED SPI | `0x01102000` | 39 |
| AR8030 SDIO host | `0x01b00000` | 48 |
| Removable SD host | `0x01c00000` | 49 |
| AXI DMA | `0x08800000` | 63, 64, 65 |
| Wave5 codec | `0x0a080000` | 68 |
| VO display | `0x08810000` | 70 |
| USB DWC2 | `0x08000000` | 74 |
| Scaler | `0x08840000` | 75 |
| QSPI NAND controller | `0x01e00000` | 98 |

Wave5 reuses the `ti,j721s2-wave521c` driver match and its firmware pathname;
that string does not identify a TI application SoC or a TI firmware image on
this board. The node explicitly requests a 500 MHz codec clock. The separate
media gate remains boot-managed in this configuration. Its SRAM window is
`0x001f0000–0x00200000`, 64 KiB; extending it to the older 128 KiB size would
overlap GIC registers beginning at `0x00201000`.

The legacy `ar_mpp` node also lists SPI 68 alongside JPEG 69 and GE2D 79.
It remains `okay`, but its codec IRQ forwarder and the Wave5 driver are
alternative owners, not evidence of simultaneous operation. The board's later
V4L2/DRM runtime retired these MPP compatibility modules.

The legacy MPP child and the active top-level AXI-DMA node both describe
`0x08800000`. The child is retained as lookup metadata beneath a parent that is
not `simple-bus`; it must not be counted as a second physical DMA engine.
The active node requests three channels, one master, 64-bit data-width encoding,
4096 block-size entries and priorities 0/1/2. The source labels some geometry
choices conservative; declaration alone does not measure the hardware maximum.

ADC, temperature and clock-controller windows overlap at `0x0a108000` by design.
Their compatible drivers must coordinate accesses; duplicate MMIO description
does not grant independent ownership. The fixed 150 MHz clock supplies several
peripherals, with AXI DMA and DSI register clocks explicitly using it as a
stand-in. This is not a complete clock/reset/power-domain model. The external
DS1307 at I²C address `0x68` is declared; the SoC-internal RTC is not.

## Display graph and board policy

The compiled graph connects VO → DSI → QY45043A0 with reciprocal endpoint
references. VO's window is 64 KiB and DSI's is 4 KiB, keeping their register
claims separate from neighboring scaler and MPP windows. The panel references
GPIO43 reset, GPIO95 enable and PWM0 channel 1 for its backlight.

GPIO hogs appear in the order 42 low, 107 low, then 100 high, expressing the
scan-strap-before-power requirement. GPIO100 supplies both panel power and SD
I/O; the SD node has no independent `vqmmc-supply`. Both MMC nodes declare a
100 MHz controller clock/range ceiling, which does not contradict the recorded
50 MHz SD card clock. The tree alone does not establish actual bus throughput.

The LED SPI child uses `rohm,dh2228fv` to obtain a userspace SPI binding; it is
not evidence of that named part on the board. USB remains declared OTG, with
vendor-derived gadget FIFO sizes. Neither the compatible string nor these FIFO
properties specify the running USB function composition. Keypad thresholds,
backlight levels and analog calibration are covered by the
[analog/PWM study](../soc/ar9311-analog-and-pwm.md).

## Offline verification

The source preprocessed and compiled with `cpp` and `dtc`, using installed
Linux 6.14 interrupt/GPIO binding constants. Compiled-property checks verified
reservation adjacency, the shared media phandle, partition continuity, eight
OPPs, reciprocal display endpoints, GPIO hog ordering and selected IRQ/clock
relationships. This was not a target Linux 6.18 build or a binding-schema run.

Compilation retained three warnings: two legacy MPP DMA children have register
properties without unit addresses, and ADC/temperature share a unit address.
Those warnings are recorded rather than suppressed. Successful compilation
does not resolve driver binding, shared-register concurrency, actual clock
rates, flash contents or reset-time memory retention.
