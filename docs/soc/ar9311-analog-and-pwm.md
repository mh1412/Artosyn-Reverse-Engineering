# AR9311 ADC, temperature and PWM interfaces

[SoC studies](README.md) · [VR04 reference board](../devices/betafpv-vr04-reference.md)

## Scope and ownership

These register/API findings come from the retained MissingLynk kernel snapshot
`e3885c6394a7025eacd3a23f473276d9823beb25`, vendor-driver reconstruction and
reported VR04 bench results. The ADC and protemp source implementations were
also read directly. Board wiring, calibration constants and UI policies below
are not Gnd485 or SC231AI P1 measurements.

SAR ADC and protemp share base **`0x0a108000`**, but are separate sampling engines.
ADC claims a `0x21c` window; protemp maps an overlapping `0xd4` window without an
exclusive resource claim. Both update control offset 0 through read/modify/write:
ADC bit 7, protemp bit 0. Their separate per-driver mutexes do not establish a
shared cross-driver lock. Recorded coexistence is not a general concurrency proof.

## SAR ADC registers and sampling

The recovered AR9311 path exposes eight channels with 10-bit conversion codes,
read through a 16-bit data value.

| Offset | Role |
| --- | --- |
| `0x00` | ADC enable bit 7 |
| `0x18` | Sampling gate: 0 begins a burst; 1 holds after sampling |
| `0x24` | Channel index, masked to eight bits |
| `0x2c` | Calibration enable bit 3 |
| `0x200` | Data reset, written 0 at initialization |
| `0x210` | Data read |

The vendor selects a different layout for an MMIO resource larger than `0x80f`:
the `9341` path uses reset/data offsets `0x800`/`0x810`. The inspected open driver
implements the AR9311 offsets directly; recognizing the vendor branch does not
establish that the open driver supports both variants.

For each read it selects the channel, writes sampling gate 0, discards 16 warm-up
reads, averages the requested samples and writes gate 1. The vendor ADC uses a
sorted, eighth-trimmed mean when sample count exceeds three; the open ADC uses
a **plain mean**. Its fallback sample count is one, but the inspected goggle DT
sets **four**. Earlier prose that assumed the default was always active is stale.

At probe, the driver initializes the engine, enables calibration per channel,
samples its reference code and clears calibration. It exposes raw and processed
values plus a gain-only scale. Invalid reference codes (`<=0` or `>=1024`) fall
back to the gain-only line.

## Voltage units and calibration

For valid per-channel reference `r`, the inspected open implementation returns:

```text
processed_mV = 894 + round((code - r) × (2113 - 894) / (1024 - r))
fallback_mV  = code × 2113 / 1024
scale        = 2113 / 1024 mV per code
```

**Processed output is integer millivolts** in this implementation; button DT
thresholds are microvolts. The source report's description of the vendor's
microvolt computation must not be copied as the open driver's return unit.
RAW×SCALE omits the reference offset and is not equivalent to PROCESSED.

The reference work changed nominal full-scale/reference constants 2025/900 mV
to 2113/894 mV after battery-channel measurements. Its fitted pin-voltage line
was approximately `2.043 × raw + 21.74` mV. Losing that offset produced about
22 mV pin error, or 0.46 V at the reported roughly 20.7:1 pack divider.
Three pack-voltage points were 6.0/7.1/8.4 V, with reported residual below 16 mV.

Full-scale and divider ratio trade off when inferred only from pack voltage;
independently measuring the divider is needed to separate them. The constants
are compiled into this driver, not DT properties, so claims that every board
port requires only a DT edit are too broad. These are reference-board calibration
values, not guaranteed SoC-wide accuracy.

## Goggle keypad ladder

The eight-button ladder uses ADC channel 0, 10 ms polling and key-up threshold 0.
The retained device tree has these thresholds and Linux key codes:

| Button | Key code | Threshold, µV |
| --- | ---: | ---: |
| Bind | 73 | 200000 |
| Back | 66 | 399968 |
| Record | 77 | 601024 |
| Up | 87 | 800000 |
| Left | 65 | 1000000 |
| Right | 68 | 1200000 |
| Enter | 69 | 1398016 |
| Down | 83 | 1550848 |

The unusual escaped DTS text for Up decodes to bytes `00 0c 35 00`, or 800000;
it is not a different threshold. The reference air unit instead has a polled
GPIO42 bind button. Button count, thresholds and GPIO wiring belong to the board.

## Protemp engine and compatibility ABI

| Offset | Role |
| --- | --- |
| `0x00` | Enable bit 0 |
| `0x38` | Set bits `0x70`; exact analog-mode meaning unresolved |
| `0xcc` | Low eight temperature-code bits |
| `0xd0` | Bit 0 supplies code bit 8 |

The free-running sample is `(LO & 255) | ((HI & 1) << 8)`. There is no SAR
sampling gate or 16-read warm-up. Despite the channel metadata's 12 real bits,
the actual reader assembles **nine bits**. The goggle DT selects eight samples;
the driver caps the request at 64.

Conversion is signed integer division of `(raw × 5320 - 1373400) / 10000`,
returning whole °C: code 300 → 22, 320 → 32, 355 → 51, 400 → 75. Matching the
vendor conversion does not validate absolute sensor accuracy. Signed behavior
below zero is an open-driver correction to the vendor's unsigned arithmetic,
not a measured subzero qualification.

For compatibility, `in_temp_scale` contains the **entire temperature in °C**,
not an ordinary IIO scale factor. RAW remains available; there is no PROCESSED
or OFFSET interface. Userspace discovers the IIO device by `name=temperature`,
not a fixed `iio:deviceN` directory.

The simple vendor protemp driver uses a plain mean. The open driver intentionally
borrows sorted trimming from a different vendor temperature driver: for more
than three samples it drops `floor(N/8)` at each end. At N=8, six middle values
remain. At N=4–7, no values are discarded. The sibling `artosyn,temperature`
implementation has different clocks, IRQ/buffering and a 12-bit table-based
conversion; it is not instantiated by this goggle's device tree.

## PWM provider and board policies

Two eight-channel controllers use a 150 MHz reference clock: `0x01000000` for
the goggle backlight and `0x01002000` for the buzzer. Per channel N:

| Register | Offset | Meaning |
| --- | --- | --- |
| OFF count | `N × 0x14` | Low interval in clock cycles |
| CTRL | `N × 0x14 + 8` | Enable mask `0x0b` |
| ON count | `0xb0 + N × 4` | High interval in clock cycles |

Counts use `round(time_ns × clock_Hz / 1e9)`; period is OFF+ON. The custom
provider exposes PWM operations; backlight uses a standard PWM consumer, while
userspace retains the buzzer's duty-based volume policy.

| Consumer | Channel | Recorded policy |
| --- | --- | --- |
| Backlight | Controller `0x01000000`, channel 1 | 500000 ns period; duty `level × 40000 + 100000` ns for levels 1–10, or 28–100% |
| Buzzer | Controller `0x01002000`, channel 0 | 260000 ns period, approximately 3.85 kHz; duty `volume × 13000` ns for volume 1–10 |

Stock backlight disable writes duty zero while leaving enable set. Buzzer volume
10 is 50% duty; the reference chose userspace PWM rather than a fixed-duty beeper
interface so it could preserve volume control. Recorded tests verified PWM state
round-trips and an audible increase through the ten volume steps.

Vendor `pwmchip8` naming is a legacy allocation result. The open reference found
the buzzer as `pwmchip1`, but probe order can change that. Discover the controller
through its platform device `1002000.pwm`; the chip number is not a board address.
