# G11 battery measurement and service-input reconstruction

[SoC studies](README.md) · [G11](../devices/skydroid-g11.md) · [ADC acquisition](g11-gd32-acquisition.md)

## Stock charge status and pin roles

Stock 0.5 code at `0x080086b8..0x080086dc` writes byte 21 of its 23-byte
status packet according to PF7/PB11. The SDK independently names the values:

| PF7 | PB11 | Value | Meaning |
| --- | --- | --- | --- |
| 0 | 0 | 0 | Not charging |
| 0 | 1 | 1 | Charging |
| 1 | Either | 2 | Full |

PF7 has priority. A custom-firmware SWD read observed PB11 high and PF7 low,
consistent with the charging branch; it was not a charging-current measurement.
Voltage, estimated percentage and charge-pin validity are separate. Unknown
percentage must not hide a known charging state, and the highest percentage
bucket does not imply the charger has signaled Full.

Stock tracing also corrected an early map that treated service pins as RC
switches:

| Pins | Software role | Qualification |
| --- | --- | --- |
| PC14 / PC15 | First three-position input | First low → 0, both high → 1, second low → 2 |
| PB2 / PB12 | Second three-position input | Same encoding |
| PF6, PB4, PA8 | Three independent button-handler inputs | Physical labels still require confirmation |
| PC13 | Active-low power button | Separate from RC mapping |
| PB11 / PF7 | Charging/full inputs | Excluded from RC mapping |

The provisional custom map used SA/SB for the pairs and SC/SD/SF for the three
buttons. Both contacts low is rejected rather than silently selecting one
position. Stock PC13 releases after 2–79 ticks set the short-event flag at
`0x20001cb7`; reaching 80 held ticks sets the long-event flag at `0x20001cb8`.
These are stock tick decisions, not measured milliseconds or a complete
shutdown sequence. See [power control](../firmware/g11-power-and-shutdown.md).

## Voltage estimate and hysteresis

The custom voltage conversion uses the internal reference and battery ADC
channel described in [acquisition](g11-gd32-acquisition.md). Its absolute
accuracy has not been physically calibrated. The historical Linux backend
applies the stock percentage thresholds to a mean of four distinct accepted
voltage reports. At its normal 2 Hz reporting cadence this takes about 1.5
seconds to initialize; redraws do not advance it. New sessions or gaps of at
least two seconds rebuild the history.

| Level | Initial selection | Downward transition | Upward transition | Display |
| --- | --- | --- | --- | --- |
| 0 | Below 3451 mV | — | ≥3451 mV → 1 | ~10% |
| 1 | 3451–3600 mV | <3400 mV → 0 | >3600 mV → 2 | ~25% |
| 2 | 3601–3750 mV | <3550 mV → 1 | ≥3751 mV → 3 | ~50% |
| 3 | 3751–3900 mV | <3700 mV → 2 | ≥3901 mV → 4 | ~75% |
| 4 | ≥3901 mV | <3850 mV → 3 | — | ~100% |

Stock evaluates branches from level 4 down to 0: a falling update can cross
several levels, while a rising update advances at most one. Initial selection
and transitions matched 30,012 executions of the saved ARM code over integer
voltages 0–5000 mV and uint16 maximum. The four-report filter does not reproduce
stock's 100-result averaging cadence or charge-dependent ADC compensation.
The tilde marks a coarse voltage estimate, not measured capacity or state of
charge accuracy.

## Low-voltage warning

Installed stock 0.5 sets its warning below **3390 mV** and clears it at or above
**3431 mV** (`0x08007baa..0x08007bd6`, state byte `0x20001cc4`). Comparison
against the actual ARM branches covered 10,002 combinations of integer voltage
and previous warning state. The earlier 3360 mV observation belongs to the
downloaded 0.8 image and must not replace the installed 0.5 result.

The historical custom UI used four valid distinct reports, displayed a red
warning over every page including transparent video, and woke a blanked screen
once when entering low voltage. The banner did not intercept touch or stop
mixing. Stale/invalid voltage cleared the filter and, after prior valid data,
showed “Battery data unavailable”; four new reports rebuilt status. Charging
alone did not clear a low-voltage warning.

Its optional 100 ms tone on warning entry and at most once per ten seconds
respected silent mode. Later autonomous firmware owns local warnings and
arbitrates them with timer/trim tones. These alerts are not battery protection:
they do not establish automatic shutdown, calibrated discharge thresholds or
measured actuator behavior. See [feedback and audio](../firmware/g11-feedback-and-audio.md).

The stock comparison image was the saved 64 KiB read with SHA-256
`e5773bae5ca64232b0536c1589d21ac87bbd32057f04b8b17aba02ec63bfc402`.
That is a different-sized artifact from the later complete 128 KiB stock backup.

## Autonomous transmitter-voltage source

Capability revision 8 introduced EdgeTX `TX_VOLTAGE` from MCU-local measurement,
without a Linux response in the mixer path. The handset service samples every
100 ms; the engine reads the latest valid voltage before each mix. Units are
0.1 V, truncated from mV: 3899 mV becomes 38. This is neither percentage nor
the four-sample warning estimate. Ordinary source scaling applies, so a direct
voltage mix is a small channel value.

The sample must be voltage-valid, 2000–5000 mV and less than 500 ms old. Invalid
reference ADC readings do not invalidate ordinary stick acquisition. Invalid
or stale voltage produces zero with source validity false: input lines yield
to fallbacks, logical predicates become false before normal duration/delay
processing, source-driven GVars retain their values, and direct mixes/weights
read zero. Warm model activation rebuilds the cached source; RF inhibition
does not hide battery data.

Signed-source corrections preserve native voltage units for negative logical
thresholds and classify negative numeric sources as GVars only if their
absolute index is inside the GVar range. This also prevents out-of-range access
for inverted battery weights.

On boot 112, SWD observed 3853–3857 mV and engine value 38 with RF off. A
ten-second window showed 499.90–500.29 Hz mixing; no physical model was changed
to exercise a battery-based channel rule. Combined ARM tests exercised source
validity, signed use, fallbacks, Linux loss and freshness boundaries with modeled
peripherals. They do not prove voltage accuracy or physical receiver behavior.

## Linux publication correction

The worker formerly held a received handset reply until the remainder of its
radio/timer/source batch finished. It now publishes that reply immediately.
A gated next-response test distinguishes the two behaviors, including immediate
invalidation. Request-start age, the 500 ms cutoff and sequence/session checks
remain unchanged; rereading an old sample cannot make it fresh.

The deployed UI follow-up preserved boot 115 and model values. Over 120 seconds,
245 observations retained fresh controls, sources and handset data, with maximum
handset age 299 ms and zero Linux mixer cycles. The original UI also passed its
72-sample baseline: the ordering defect is demonstrated, but attributing an
earlier isolated physical freshness lapse to it remains an inference.
