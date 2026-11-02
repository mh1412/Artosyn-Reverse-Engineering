# G11 GD32 startup, input acquisition and control timing

[Chip studies](README.md) · [G11](../devices/skydroid-g11.md) · [Memory and models](../firmware/g11-mcu-memory-and-models.md)

The G11 control MCU acquires ADC/GPIO inputs and, in the later autonomous
implementation, runs the actual EdgeTX mixer independently of Linux. The
October 23 component fixtures and October 25 physical autonomous tests have
different scopes. Early “no flashable firmware” statements do not describe
the later installed direct-boot application.

## Startup and clocks

The core vector table has 65 entries, 260 bytes, aligned to 512 bytes. USART0
IRQ27 serves the external control link; USART1 IRQ28 serves Linux. Reset entry
requires privileged Thread mode, masks interrupts, establishes MSP, clears
CONTROL/BASEPRI, stops inherited SysTick, clears pending SysTick/PendSV and
implemented NVIC banks, installs VTOR and enables aligned exception stacking.
PRIMASK stays set until initialized board code enables interrupts.

Before C data/BSS initialization, an assembly hook stops inherited DMA writers.
It clears CHEN on all seven channels and waits at most 32 reads each before
changing the remaining control bits, then resets ADC, USART0/1 and the seven
implemented timers with readback. GPIO, flash, watchdog, backup domain and
unrelated reset bits are preserved. Failed quiescence traps before SRAM
initialization without feeding an inherited watchdog. This establishes digital
ordering, not an electrical guarantee that peripheral reset leaves every pin
voltage unchanged.

UART IRQ setup gives both lines equal priority 8 of 16, preventing mutual
preemption regardless of inherited PRIGROUP. It clears pending state again after
drivers initialize. Core startup does not by itself establish output shutdown,
power hold or recovery from arbitrary inherited NMI/watchdog states.

The clock routine preserves IRC8M trimming, switches to IRC8M before stopping
the PLL, clears multiplier-extension bits, then selects IRC8M/2 × 12: nominal
48 MHz SYS/AHB/APB1/APB2. One shared register-read budget bounds readiness waits;
it is not a measured timeout. Failed setup returns zero rather than an assumed
clock rate. External frequency accuracy was not measured in this study.

TIMER1 is the device's 32-bit timebase; TIMER2 is 16-bit. The driver configures
an integer prescaler for 1 MHz, loads it with an update event and runs with
all-ones auto-reload, no interrupt/DMA/capture outputs. One counter read supplies
wrapping microseconds. This reassigns the timer used by stock trainer generation.
The caller must account for APB timer doubling and avoid resetting/changing this
clock during a session.

## Independent watchdog

IRC40K drives the watchdog independently. Startup waits for oscillator/register
synchronization, rejects an inherited window other than `0xfff`, waits 10 ms
before first reload and configures /16 with reload 2500. The resulting timeout
is nominally about one second, not a measured worst-case bound. Runtime feeds
are throttled to once per 10 ms to respect oscillator-cycle requirements.

Feeding belongs to a progressing foreground loop. A stopped core timebase cannot
advance its feed timestamp, allowing the independently clocked watchdog to
expire. An already-running short watchdog may reset during startup; no readable
enable bit or this driver can guarantee disarming it. The later direct-boot
profile restarts its application after fault reset; the earlier resident-loader
workflow is historical. An unbootable application uses SWD recovery.

## Raw inputs and DMA publication

These are MCU pin/controller identities, not connector pin numbers or a fully
measured stick-order map:

| Pins | Mode | Published validity |
| --- | --- | --- |
| PA0, PA1, PA4, PA5, PA6, PA7, PB0 | Analog, no pull | ADC channels 0/1/4/5/6/7/8, mask `0x01f3` |
| PA8 | Input pull-up | Port A mask `0x0100` |
| PB2, PB4, PB11, PB12 | Input pull-up | Port B mask `0x1814` |
| PC13, PC14, PC15 | Input pull-up | Port C mask `0xe000` |
| PF6, PF7 | Input pull-up | Port F mask `0x00c0` |

Only listed mode/pull fields change. Output values, alternate functions, drive
settings and unrelated pins are preserved. Raw physical mapping began as a
provisional bring-up map; calibration and electrical sampling are separate.

With actual APB2 at 48 MHz, APB2/8 gives nominal 6 MHz ADC operation. Seven
external 12-bit channels use 71.5 sampling plus 12.5 conversion cycles; VREFINT
channel 17 uses 239.5 sampling cycles. Total configured scan time is about
140 µs, calculated rather than measured. Source impedance, settling and absolute
accuracy remain unqualified.

Startup refuses an enabled ADC, enabled DMA0 or remapped ADC request before
reset/configuration. It preserves other clocks and never resets the entire DMA
controller. Board startup must still ensure no other peripheral owns a shared
DMA request. Power-on, warmup, calibration reset, calibration and scanning are
separate bounded foreground states, with 10 ms calibration timeouts.

Each scan timestamps before sequential GPIO capture and ADC trigger. It is not
a simultaneous sample of all controls. DMA0 reads the 32-bit ADC register into
eight SRAM halfwords with increment/high priority, no interrupts or circular
mode. Only full-transfer completion **and zero remaining count** permit
publication. DMA is disabled before buffer inspection. Values above 4095 fault;
partial overwrites never publish. EOC is unsuitable because DMA reads can clear
it; this part has no ADC-overrun flag to substitute.

Scans start at least 1 ms apart. An incomplete scan at 1 ms faults. A complete
scan can be collected later only while its original trigger age is below 2 ms;
collection does not refresh its timestamp. The last complete fresh scan may be
returned while a new scan is pending. Fault stops ADC/DMA and invalidates output.

An October 25 fault occurred at 1,001 µs despite complete DMA flags, zero remaining
count and valid data. The earlier cutoff had confused late foreground collection
with incomplete conversion. The corrected driver distinguishes them, retaining
the 2 ms age limit and recording completion age/late counts. Production scheduling
still must protect acquisition from optional Linux traffic.

The first captured INPUT failure could not distinguish timeout from DMA error:
shutdown had already cleared the flags. It also exposed a second problem: a
pending USART1 interrupt could repeatedly re-enter while foreground shutdown
was unfinished. Terminal-state IRQ handlers were changed to disable their UART
source. In the next diagnostic capture, after about 319.8 seconds, the retained
values were elapsed 1,001 µs, DMA flags 7, remaining count 0 and SCAN_TIMEOUT.
Both UARTs were disabled and the core was in Thread mode, establishing that
shutdown completed in that observed failure.

The corrected driver passed 60 recorded ARM register-model scenario groups;
the old driver failed the completed-at-1,000-µs regression. On hardware, a
169.95-second observation of image `cae337ed…` accepted two naturally late
completed scans at 1,000 µs and continued for 60 seconds after the first event,
with the same MCU nonce and no observed runtime/UART fault. This supports the
specific collection-time correction. It does not establish long-run reliability,
explain every earlier reset, or qualify autonomous mixing and receiver output.
The separate [session timestamp defect](../firmware/g11-uart-and-session-lifecycle.md#long-session-timestamp-defect)
affected admission of fresh data after approximately 35.8 minutes.

Fault diagnostics preserve first reason, phase elapsed time, pre-clear DMA flags
and remaining count. Codes 1–6 identify calibration timeout, DMA error, incomplete
scan, inconsistent count, invalid value and stale completion. Session freshness
can independently produce an input fault; driver state and runtime state both
matter. A stalled timebase requires watchdog recovery, not software timeout alone.

## Reference-compensated battery estimate

The same completed scan retains ADC8 and internal ADC17 for the power API:

```text
VDDA_mV = floor(1212 × 4095 / ADC17)
battery_mV = floor(VDDA_mV × ADC8 × 2 / 4095)
```

The integer operation order and 1212 mV assumption match the examined stock
firmware. Zero reference, estimated VDDA outside 2.6–3.6 V, stale data or stopped
acquisition return unavailable without changing the caller's output. An unusable
reference alone need not invalidate healthy controls; corrupted DMA values still
fault acquisition. Channel 17 stays separate from external raw-control fields.

Live SWD reads observed ADC17 around 1498–1500, estimated VDDA about 3.31 V and
battery 3.96–3.97 V. Those reads were not atomic and had no multimeter comparison.
This is a voltage estimate, not calibrated state-of-charge. Charger state uses
separate PB11/PF7 interpretation. A 90.22-second historical input soak received
8,942 new samples with one UART overrun and one decoder rejection, RF off;
it was not loss-free transport.

## Autonomous scheduling and Linux independence

The installed October 25 profile uses a 2 ms mix period, 1.5 ms execution budget
and 1 ms permitted start lateness. Inputs are serviced around management work;
an iteration handles at most 16 received bytes/one packet. Full validation and
flash mutation require acquisition/control/RF stopped. Legacy Linux channel-output
packets do not drive this autonomous mixer.

Startup claims a durable boot nonce, validates the exact journal-selected model
and begins acquisition/mixing without requiring Linux. A newer staged model is
not implicitly selected. Missing/corrupt selection leaves control stopped.
Linux UART faults permit bounded peripheral recovery at 100 ms intervals while
preserving model and mixer state. Input, external-control or scheduler faults
instead stop control/output and watchdog feeding. These failure domains are
deliberately separate.

The first physical RF-off Linux reboot changed the Linux boot ID while the MCU
boot nonce and active model stayed unchanged: 137 samples over 29.94 seconds
showed average output progression 500.023 Hz. A separate MCU reset restored the
selected model before the first 189,408 µs observation; 21 samples over 4.5 seconds
averaged 500.124 Hz before UI reconnect. Only nonce flash changed. That MCU reset
also restarted Linux, which subsequently restored Wi-Fi/UI.

These observations establish the exercised reset paths, not receiver continuity
or physical button/off/wake behavior.

## Timing measurements and their limits

The first simple-model build's observed mix peak was 1,396 µs against its
1,500 µs budget. Profiling found periodic traversal of all 64 logical-switch
slots across nine flight modes even though the model configured none. A
configured-index optimization reduced measured peaks:

| Window | Before | After |
| --- | ---: | ---: |
| UI stopped | 1,173 µs | 624 µs |
| UI connected | 1,402 µs | 759 µs |
| Headroom to budget, UI connected | 98 µs | 741 µs |

A subsequent Linux reboot observed about 500.019 Hz and a 760 µs peak. A separate
active-CRSF counter window observed 500.006 Hz mixing, 3,665 channel transmissions
over 15.232 seconds and an 840 µs peak, but its analyzer inputs stayed high and
no module replies were received. Those counters did not establish physical
wire delivery. Later electrical investigations need their own evidence.

The profiler includes IRQ/profiling overhead. SWD reads are also non-atomic:
at 400 kHz, a bracket around one output-counter read took as long as 4,207 µs,
longer than a mix period. An apparent 477 Hz short-window result from separately
read clock/counter values was superseded by bracketed ten-second bounds of
499.83–500.21 Hz. Neither average-rate bounds nor observed peaks are worst-case
latency guarantees. Heavier models, external-clock accuracy, physical output,
power sequencing and endurance remain separate qualification work.
