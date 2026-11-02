# AR9311 platform registers and ownership

[SoC studies](README.md) · [AR9311 overview](ar9311.md) · [Platform experiments](../experiments/ar9311-platform-validation.md)

The shared register geometry below comes from the reconstructed device tree and
driver work. Board wiring and validation refer to Ascent Gnd485 unless explicitly
labeled as reference-platform findings. A node or driver implementation does not
establish that every peripheral was exercised on every board.

## Register and interrupt map

Interrupt numbers here are GIC SPI numbers from the device tree, not Linux's
dynamically assigned IRQ numbers.

| Block | Base / extent | SPI |
| --- | --- | --- |
| Codec SRAM | `0x001f0000`, 64 KiB | — |
| GIC distributor | `0x00201000`, 4 KiB | — |
| I²C 0 | `0x01200000`, 8 KiB | 32 |
| I²C 1 | `0x01202000`, 8 KiB | 33 |
| UART 0 | `0x01500000`, 1 KiB | 11 |
| DW watchdog | `0x01600000`, 4 KiB | 37 |
| Radio SDIO host | `0x01b00000`, 4 KiB | 48 |
| Removable SD host | `0x01c00000`, 4 KiB | 49 |
| QSPI | `0x01e00000`, 4 KiB | — |
| DWC2 USB | `0x08000000`, 256 KiB | 74 |
| VO display | `0x08810000`, 64 KiB | 70 |
| DSI | `0x08850000`, 4 KiB | — |
| VIF | `0x08870000`, 64 KiB | 62 |
| CSI | `0x08880000`, 64 KiB | 60, 61 |
| ISP | `0x08c00000`, 2 MiB | 71 |
| CVISP | `0x08e00000`, 64 KiB | — |
| Wave521C codec | `0x0a080000`, 64 KiB | 68 |
| GPIO | `0x0a10a000`, 8 KiB | No irqchip in the examined GPIO driver |

Codec resources additionally include media CRG at `0x0a106000`, isolation at
`0x0a102000` and codec clock at `0x0a104018`. VO uses pixel-clock control
`0x0a10404c` and DC CRG `0x0a106018`. UART registers are four bytes wide with
register shift 2. Gnd485 retains firmware-established console timing at
1,152,000 baud and uses PSCI for its two Cortex-A53 cores.

The Gnd485 RAM layout reserves secure firmware at `0x23400000–0x23ffffff`,
ramoops at `0x2ff00000–0x2fffffff` and codec DMA at `0x30000000–0x3fffffff`.
Linux's declared lower 256 MiB includes the secure and ramoops reservations.
These are board allocations, not universal AR9311 RAM sizes.

## GPIO registers and numbering

Each bank starts at `0x0a10a000 + 0xbc + bank × 0x0c`:

| Bank-relative offset | Meaning |
| --- | --- |
| `+0` | Output-value latch; updates use a software shadow |
| `+4` | Direction; bit 1 means input, bit 0 means output |
| `+8` | Physical input readback |

The first bank offset is **`0xbc`, not `0xc0`**. Using `0xc0` as a bank base
makes an intended output write hit the direction register. Pinmux occupies the
earlier `0x00–0x3c` region and remains an early-firmware responsibility in this
driver.

| Bank | Global first line | Line count | Global range |
| --- | --- | --- | --- |
| 0 | 0 | 23 | 0–22 |
| 1 | 23 | 22 | 23–44 |
| 2 | 45 | 26 | 45–70 |
| 3 | 71 | 6 | 71–76 |
| 4 | 77 | 6 | 77–82 |
| 5 | 83 | 11 | 83–93 |
| 6 | 94 | 16 | 94–109 |

The driver registers one gpiochip per bank. Device-tree references retain vendor
global numbers; a custom translator selects the bank and subtracts its base.
The examined interface supplies get/set/direction operations, without GPIO IRQ
support. Polled input consumers are therefore distinct from interrupt-driven
button support.

### Direction-readback correction

Early reference notes asserted that direction registers were unreadable and
initialized the direction shadow to all inputs. Gnd485 measurements contradict
that assertion. Cold U-Boot bank 0 read `0x007fbfff`, preserving output 14.
Linux bank 0 read `0x007fa9f7`, matching outputs 3/9/10/12/14; bank 1 read
`0x003bcfff`, matching local outputs 12/13/18; bank 6 read `0x0000dfff`, matching
local output 13. All seven banks had meaningful readback.

The matched vendor GPIO probe passes a direction address and flags zero to
`bgpio_init`, which reads that register into its shadow. Vendor suspend code
also reads bank directions, although this does not prove suspend was tested.
The inherited unreadability assertion had no accompanying raw failure or erratum.
An earlier offset error is a possible explanation, not an established cause.

The final source change uses `artosyn,ar9311-gpio` in the shared SoC description
to seed direction shadows from hardware before registering consumers. The older
`artosyn,gpio` fallback keeps its prior behavior for unidentified variants.
This supersedes the initial Gnd485-specific compatibility switch. Source/module
and DT validation covered the generalization; a new full-image hardware test
was not part of that particular change.

Direction readability does not make output-latch read/modify/write equivalent
to physical input reads. Output-shadow semantics remain unchanged. Preserving
boot directions prevents the first consumer from releasing unrelated firmware
outputs; switching glitches were not measured with a scope.

## Gnd485 GPIO consumers

| Global line | Board function and configured behavior |
| --- | --- |
| 3 | AR8030 reset, active low |
| 9 | USB device/debug-mode selection, high |
| 10 | USB debug selection, low |
| 12 | Radio 3.3 V supply enable, high; 100 ms startup delay |
| 14 | SD power enable, high |
| 35 | Radio I/O voltage selection, high for 3.3 V |
| 36 | SD I/O voltage selection, high for 3.3 V |
| 41 | Upstream radio power enable, high; 100 ms startup delay |
| 106 | HDMI bridge interrupt noted in wiring; Linux uses polling |
| 107 | HDMI bridge reset, configured active high |

These assignments differ from the reference goggle, whose radio reset is line
23 and panel uses lines 42/43/95/100/107. Reference air-unit line 42 is a bind
button. Shared line numbers do not imply shared board functions.

## MMC source clocks

The focused common-clock provider owns two source paths and their FIX/SAMPLE/DRIVE
gates. It does not own CPU, DDR, USB, codec or display PLL sequencing.

| Path | SEL / CFG addresses | Gnd485 SEL / phase |
| --- | --- | --- |
| Radio SDIO | `0x0a108088` / `0x0a10808c` | `0x87` / 2 |
| Removable SD | `0x0a1080c0` / `0x0a1080c4` | `0x80` / 0 |

SEL bits 7:6 encode source rate as `25 MHz << tap`, yielding 25/50/100/200 MHz.
Only 100 MHz sources were hardware-tested in this Gnd485 work. Card clocks are
separately divided and capped at 50 MHz, four-bit width and 3.3 V, without 1.8 V
switching. CFG updates replace only the low five phase bits.

| Gate word | Bits owned by this provider |
| --- | --- |
| `0x0a104020` | Radio bit 28 |
| `0x0a104024` | Radio bits 12 and 28 |
| `0x0a104028` | SD bits 12 and 28 |
| `0x0a10402c` | SD bit 12 |

Locked read/modify/write preserves adjacent MIPI/eMMC fields. Shared PLL power
at `0x0a108000` is untouched. Old reference comments named bits 22/23, but the
examined glue did not actually perform those gate writes; the leaf table and
hardware recovery evidence support bits 12/28.

The host wrapper requires a real CIU clock rather than falling back to a static
frequency. Clearing these gates before boot made the old image fail both MMC
resets; the new clock-owning image restored the exact gate words and recovered
SD and radio. Unbinding only the SD host disabled its clocks while leaving the
radio registered without another firmware upload.

## Watchdog and retained crash records

The DW watchdog uses a 24 MHz input and can be left running by boot firmware.
Its original Linux handover had a nominal 11-second timeout. Software can report
inactive before userspace opens it while the watchdog core keeps hardware fed.
Magic close with a five-second setting survived 15 seconds through kernel
feeding, while hardware still reported time remaining. This was not disarm.

Stopping feeds with the descriptor open reset the board into SPL/U-Boot without
power-supply intervention. No verified Gnd485 reset-controller binding stops the
counter; halt/poweroff can therefore lead to another reset after feeds cease.

A later AR9311 restart quirk, motivated by P1 recovery, selects a timeout of at
least one second using the actual clock/table rather than the minimum TOP.
The minimum repeatedly reset P1 before stock USB recovery could start. This
changes restart timing and does not implement watchdog disarm.

The 1 MiB ramoops reservation provides 128 KiB kmsg records, 256 KiB console and
128 KiB pmsg. It is outside secure firmware, load destinations and codec DMA.
Warm reboot, panic and watchdog-reset experiments retained markers; cold power
loss did not. Explicit archival copies records and checksums to SD without
deleting their originals. This is crash evidence, not persistent RAM across
power loss.

## RTC investigation

The vendor RTC functions match the X-Gene/DW register interface. A candidate
driver registered an RTC device, but setting time left the seconds counter at
offset `0x00` at zero. Load register `0x08` read back the requested epoch.
Control `0x0c` bit 2 stayed clear; version `0x1c` read `0x3230372a` and
prescaler `0x20` read `0x8000`. Offsets `0x40–0x5c` aliased `0x00–0x1c`.

The enable bit can be read-only on an always-enabled IP configuration, so its
readback alone did not diagnose the fault. Missing RTC input clock or another
board prerequisite remained unresolved. Alarm testing was not reached, the
candidate was removed from the default profile and no working RTC/retention
claim follows from device registration.

## Reference-platform clock work

Separate reference-goggle work modeled 49 read-only CGU leaves plus writable
CPU and pixel clocks. Leaf muxes use bits 10:8 or 26:24 with gates 12/28; dual-bank
leaves use bit 31 to select the active bank. A CPU composite at `0x0a100008`
mirrors `0x0a100408`; its rate change programs the idle bank before switching.
This is not evidence of validated Gnd485 DVFS or shared board power policy.

The reference pixel PLL uses a period word at `0x0a108410`, with larger words
giving slower clocks. The driver limits accumulated change to ±5% of the boot
word and slews approximately 0.05% per frame. That broad clamp exceeds the
tested DSI FIFO tolerance: approximately +0.10% produced continuous overrun,
whereas percent-scale slowing produced a quarter-line rotation without that
error latch. Restoring boot rate recovered the panel in those experiments.

The reference pacing servo could drift across restarts because DSI setup retained
boot-derived line timing while the pixel clock changed. Narrowing the clamp
preserved correct panel output but raised measured receive-to-flip median from
about 21 ms to 41–46 ms and increased repeats. These are reference-panel tradeoffs,
not Gnd485 HDMI latency results. General clock/reset/pinctrl ownership, GPIO IRQs,
thermal monitoring and board-specific power management remain separate work.
