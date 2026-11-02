# G11 local feedback, presentation events and Linux audio

[Firmware](README.md) · [G11](../devices/skydroid-g11.md) · [Battery/service inputs](../soc/g11-battery-and-service-inputs.md)

## Routes and evidence

Stock firmware configures PA11 AF2 as TIMER0_CH3 at 2 kHz: 48 MHz clock,
prescaler 47, period 499. Compare values 8, 32 and 96 implement three duty
settings. PB10 participates in the stock battery-alert pattern, but its load
and motor routing remain unqualified; custom production drivers leave it alone.

The October 24 custom-firmware test observed TIMER0_CH3CV change `0 → 8 → 0`
after a diagnostic pulse request. MCU timestamps recorded 100.011 ms. Sparse
host SWD samples spanned about 90.78 ms between observed active samples; that
sampling span is not the complete pulse duration. Input-only operation remained
live and no driver fault was reported. This proves register activation/expiry,
not an electrical waveform or audible sound.

Later work moved timer and trim decisions into the autonomous MCU and added
MCU-selected speech presentation on Linux. Both routes have software and
deployment evidence, but audible speaker/buzzer confirmation and motor
actuation remain open in the reviewed records.

## Historical Linux-requested pulses

The earlier session protocol's message 23 carried compare:uint8, aux:uint8 and
duration_ms:LE16. Capability 512 advertised feedback; 1024 separately advertised
aux, which this board did not enable. All-zero payload meant stop. A request
had to follow an accepted mixer response with matching token, sequence and
sample timestamp, be fresh and belong to a live session. It neither renewed
the control deadline nor enabled RF.

The host appended at most one feedback or module-management message per mixer
response. A single pending Linux request expired after 100 ms; busy/cooldown
rejections were not retried. There was no feedback ACK, so a host sent counter
did not establish actuation. The diagnostic tone request was one compare-8,
100 ms pulse, independent of automatic-alert silent mode.

The driver initialized silently, limited pulses to 1–300 ms and required 300 ms
off between pulses. Requests could not overlap or extend an active pulse.
Maintenance, session loss, power-off and runtime faults canceled feedback.
Register mismatches parked the output and latched a feedback fault. Polling
expired pulses; the watchdog covered stalled firmware rather than a separate
tone-expiry timer. These Linux-session rules describe the historical design,
not autonomous timer ownership.

## MCU timer and trim cues

The autonomous foreground loop consumes a bounded engine event queue. Timer
evaluation, tone selection, PWM timing and cancellation do not depend on Linux.
Timer/battery patterns retain 2 kHz at compare 8:

- Inside the configured countdown interval: one 100 ms beep per second.
- Before that interval: three 120 ms beeps at 30 seconds, two at 20 seconds,
  one at 10 seconds. A configured 30-second countdown takes precedence and
  makes its 30-second event one 100 ms beep.
- Timer elapsed: one 300 ms beep, subject to the engine's audio-event filter.

Repeated beeps have 20 ms silent gaps. A pattern's deadline is its nominal
duration plus 10 ms. Late polling preserves the real silent gap and discards
a tail that no longer fits; missed beeps are not replayed. Timer/battery
patterns retain a 300 ms cooldown.

Trim cues use events from the pinned EdgeTX key/checkTrims implementation:

| Event | Nominal frequency | Duration |
| --- | --- | --- |
| Movement | `1920 + 8 × clamp(value, -128, 128)` Hz | 40 ms |
| Center | 1920 Hz | 80 ms |
| Minimum | 896 Hz | 80 ms |
| Maximum | 2944 Hz | 80 ms |

Extended trims retain their full engine value while pitch clamps. Periods round
to the nearest 1 MHz timer tick with approximately the original 8/500 duty
ratio. Quiet and alarms-only modes suppress trim sounds; no-keys/all-sounds
permit them. This work did not assign new physical buttons to trims.

Elapsed timer events outrank countdowns, which outrank trim cues; the first
equal-priority event wins. A new trim may replace another trim immediately,
but cannot take over a timer/battery pulse or silent gap. Trim completion uses
a 20 ms gap. Due battery warnings preempt trims and wait for ordinary cooldown;
clearing a battery warning cannot cancel a separately owned timer tone.
Rejected/colliding events are counted, not queued for late playback.

Maintenance, calibration, shutdown and quiet mode cancel local feedback.
Feedback register failures silence it without faulting the control loop.
Activation resets pending engine events, preventing replay after reconnect.
The eight-entry event queue never delays a mixer tick.

The timer deployment was recorded at boot 113 and trim deployment at boot 116.
The latter preserved selected model/working values and measured 499.85–500.16 Hz
mixing with RF off; cumulative observed peaks were 802 µs mixing and 238 µs
lateness. ARM tests covered actual engine events with emulated registers/time.
These establish installation and software behavior, not physical tone patterns
or worst-case timing.

## Bounded presentation transport

A separate eight-event queue carries MCU-generated presentation intent to Linux.
It drops new events on overflow, expires them after 500 ms and discards intent
on model-generation change, maintenance, calibration, pending shutdown, quiet
mode or management OPEN. Event IDs increase without wrapping during a boot;
exhaustion drops subsequent events, and drop counters saturate.

Optional service bit 16 advertises model operation 23, EVENT. The request has
only the standard management prefix and consumes at most one event. Boot,
generation, session and sequence checks apply; exact retries return the cached
reply without consuming another event. Malformed/stale requests consume none.

The reply's standard 16-byte observation header is followed by this 48-byte
body, with integers little-endian and unused bytes zero:

| Body offset | Field |
| --- | --- |
| 0–3 | Active model generation; zero when inactive |
| 4–7 | Event ID; zero means no event |
| 8–11 | Remaining lifetime, µs |
| 12–15 | Cumulative MCU queue drops |
| 16–19 | Signed event value |
| 20–23 | Kind, category, index, flags |
| 24 | Events remaining, 0–7 |
| 25–47 | Reserved zero |

LOCAL_TONE flag bit 0 means MCU ownership, not proof a physical pulse played.
Linux subtracts the full request/retry duration from lifetime, fetches at most
two events per worker pass, and bounds its own queue at eight. Expiry, wrong
model generation, disconnect, busy state, shutdown and non-increasing IDs
prevent stale playback.

The early candidate sent raw countdown intent without a UI consumer. A later
follow-up superseded that limitation: the MCU expands speech using immutable
active timer settings. Kind 5 is Number and kind 3 Duration, with zero
category/index/flags. Linux calls its language functions with those values;
it does not evaluate the editable draft timer. The native UI consumes at most
four events per GUI poll after startup.

Kind 6 carries normalized haptic intent with LOCAL_HAPTIC bit 1. Category and
index are pulse/pause lengths in 5 ms ticks. Value bits 0–4 carry pinned
PLAY_NOW/repeat flags and bits 8–10 strength 1–5. The motor backend is absent;
Linux counts this as locally owned rather than actuating anything. Explicit
haptic timer intent can survive audio quiet mode, while speech/tones are
suppressed. Model/general audio-file events remain incomplete.

Native comparisons covered 152,064 timer-expansion cases. A real UI/PTY/ARM
fixture deliberately gave the Linux draft timer 999 seconds while the MCU
selected Number(1), verifying ownership separation. Those fixtures checked
dispatch and encoded CRSF progress, not physical sound or radio reception.

## Linux PCM and deployed voice pack

The later sink feeds SigmaStar AO device/channel 0 with 32 kHz mono signed
16-bit PCM from the existing EdgeTX audio task. EdgeTX applies digital volume;
device gain is −20 dB. The sink does not initialize HAL/SYS or interpret model
settings. It takes a process lock, refuses an occupied AO device and limits
queued data to 6,400 bytes plus one 640-byte buffer, approximately 110 ms.
Waits are cancellable and expire after 250 ms; SendFrame uses timeout zero.
Failures stop writes, and shutdown clears only the sink's own configuration.

An ALSA prototype was rejected because its accepted-frame counters advanced
while downstream AO retained samples. Inspection found 53,984 queued bytes
after a nominal two-second stream had run only 1.158 seconds. The retained
direct-AO sink observes downstream queue progress. A physical silent-stream
test sent 64,000 samples, drained in 2.008 seconds and reported zero drops,
remaining queued bytes and API failures, with cleared AO attributes afterward.
Silence draining correctly does not verify speaker wiring or audible speech.

The October 26 deployment installed presentation transport, MCU speech/haptic
expansion and the PCM sink together at boot 123, preserving schema 14 and
model/checkpoint values. The recorded MCU artifact SHA-256 was
`b14c28276f5868ea238525dba6d0dd33c7939903a83ba664e318688bd813ced1`;
the initial stripped UI was
`b38e9c84de1ea63d962a4cc6bfaa920ad902e3fd2ec4ad477e15ca29d86ccfbd`.
Three seconds of UI suspension retained approximately 500 Hz MCU mixing and
clock progress with zero Linux mixer cycles.

All 747 original WAV files from the pinned English Michelle v2.12.3 pack were
installed unchanged: 16 kHz mono PCM S16LE, about 27 MB before UBIFS compression.
Archive hash, format, paths and required timer/number assets were checked;
native 16→32 kHz resampling was verified at the SDK boundary. Sound-pack version
is independent of the development EdgeTX revision. The recorded free space
after installation was 14.8 MB, a snapshot rather than an ongoing requirement.

Enabling PCM exposed duplicate native battery alarms: the simulator's zero
voltage repeatedly queued the low-battery WAV. A G11-specific patch disabled
that native evaluator while preserving the measured stock/MCU battery path.
A UI-only follow-up applied the correction without resetting the MCU. A fixture
with a distinct low-battery waveform verified its absence over the alarm
interval while MCU-selected speech still decoded. Thus earlier notes saying
the speech consumer/PCM sink were absent or candidate-only are historical;
audible output, physical haptics and general/model audio remain unqualified.
