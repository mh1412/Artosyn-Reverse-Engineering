# G11 battery measurement and service-input reconstruction

[SoC studies](README.md) · [G11](../devices/skydroid-g11.md)

## Stock charge status and pin roles

Stock 0.5 code at `0x080086b8..0x080086dc` writes byte 21 of its 23-byte
status packet according to PF7/PB11. The SDK independently names the values:

| PF7 | PB11 | Value | Meaning |
| --- | --- | --- | --- |
| 0 | 0 | 0 | Not charging |
| 0 | 1 | 1 | Charging |
| 1 | Either | 2 | Full |

PF7 has priority. Voltage, estimated percentage and charge-pin status are
separate observations.

The October 23 stock heartbeat observations illustrate that distinction. After
network access returned, three groups reported charging with screen on and a
fixed 10% estimate while the final voltage sample rose from 3377 to 3417 to
3455 mV over roughly sixteen minutes. A reported 19 W charger reading did not
measure battery current. These non-atomic cache reads had no MCU acquisition
timestamp; they show a reported charging state and upward voltage trend, not
capacity, cell health or a calibrated charging-rate measurement. Earlier failed
network reads supplied no battery evidence.

Stock tracing also corrected an early map that treated service pins as RC
switches:

| Pins | Software role | Qualification |
| --- | --- | --- |
| PC14 / PC15 | First three-position input | First low → 0, both high → 1, second low → 2 |
| PB2 / PB12 | Second three-position input | Same encoding |
| PF6, PB4, PA8 | Three independent button-handler inputs | Physical input names require a separately verified map |
| PC13 | Active-low power button | Separate from RC mapping |
| PB11 / PF7 | Charging/full inputs | Excluded from RC mapping |

Stock PC13 releases after 2–79 ticks set the short-event flag at
`0x20001cb7`; reaching 80 held ticks sets the long-event flag at `0x20001cb8`.
These are stock tick decisions, not measured milliseconds or a complete
shutdown sequence. See [power control](../firmware/g11-power-and-shutdown.md).

## Voltage estimate and hysteresis

Stock ADC conversion uses battery channel ADC8 and internal-reference ADC17:

```text
VDDA_mV = floor(1212 × 4095 / ADC17)
battery_mV = floor(VDDA_mV × ADC8 × 2 / 4095)
```

The 1212 mV reference assumption and integer operation order come from the
examined stock firmware; absolute voltage accuracy was not physically calibrated.
The recovered percentage thresholds are:

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
voltages 0–5000 mV and uint16 maximum. Stock also uses 100-result averaging and charge-dependent ADC compensation;
threshold equivalence alone does not establish the complete sampling cadence.
The tilde marks a coarse voltage estimate, not measured capacity or state of
charge accuracy.

## Low-voltage warning

Installed stock 0.5 sets its warning below **3390 mV** and clears it at or above
**3431 mV** (`0x08007baa..0x08007bd6`, state byte `0x20001cc4`). Comparison
against the actual ARM branches covered 10,002 combinations of integer voltage
and previous warning state. The earlier 3360 mV observation belongs to the
downloaded 0.8 image and must not replace the installed 0.5 result.

These thresholds do not establish calibrated state of charge or automatic
battery protection.

The stock comparison image was the saved 64 KiB read with SHA-256
`e5773bae5ca64232b0536c1589d21ac87bbd32057f04b8b17aba02ec63bfc402`.
That is a different-sized artifact from the later complete 128 KiB stock backup.
