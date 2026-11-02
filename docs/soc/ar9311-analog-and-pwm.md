# AR9311 stock ADC, temperature and PWM findings

[SoC studies](README.md) · [VR04 board](../devices/betafpv-vr04-reference.md)

These findings come from vendor-driver reconstruction and reference-board
measurements retained at kernel snapshot `e3885c6394a7025eacd3a23f473276d9823beb25`.
Board calibration and wiring are not automatically applicable to Gnd485 or P1.

## SAR ADC

SAR ADC and protemp share base `0x0a108000` but are separate sampling engines.
The recovered AR9311 ADC has eight channels and 10-bit conversion codes:

| Offset | Role |
| --- | --- |
| `0x00` | ADC enable bit 7 |
| `0x18` | Sampling gate: 0 begins a burst; 1 holds after sampling |
| `0x24` | Channel index, masked to eight bits |
| `0x2c` | Calibration enable bit 3 |
| `0x200` | Data reset, written 0 at initialization |
| `0x210` | Data read |

The vendor selects reset/data offsets `0x800/0x810` for its alternate 9341 path
when the MMIO resource exceeds `0x80f`. AR9311 uses the offsets above.
The sampling sequence selects the channel, starts a burst, discards 16 warm-up
reads, collects samples and holds the gate. Vendor ADC averaging uses a sorted,
eighth-trimmed mean when the sample count exceeds three.

Reference pack-voltage measurements fitted approximately `2.043 × raw + 21.74`
mV at the pin. Omitting the offset produced about 22 mV pin error, or 0.46 V at
the reported roughly 20.7:1 divider. The 6.0/7.1/8.4 V test points do not separate
ADC full-scale error from divider error or establish SoC-wide accuracy.

## Protemp engine

| Offset | Role |
| --- | --- |
| `0x00` | Enable bit 0 |
| `0x38` | Set bits `0x70`; exact analog-mode meaning unresolved |
| `0xcc` | Low eight temperature-code bits |
| `0xd0` | Bit 0 supplies code bit 8 |

The sample is `(LO & 255) | ((HI & 1) << 8)`: nine bits, despite metadata
claiming 12. There is no SAR sampling gate or 16-read warm-up. The recovered
conversion expression is `(raw × 5320 - 1373400) / 10000`; stock uses unsigned
arithmetic, so a signed interpretation below zero must not be attributed to it.
Examples in the positive range are 300 → 22, 320 → 32, 355 → 51 and 400 → 75 °C.
Absolute temperature accuracy was not independently calibrated.

The simple vendor protemp driver uses a plain mean. The sibling
`artosyn,temperature` implementation has different clocks, IRQ/buffering and a
12-bit table conversion. Those two interfaces must not be conflated.

## PWM hardware

Two eight-channel controllers use a 150 MHz reference clock: `0x01000000` for
the goggle backlight and `0x01002000` for the buzzer. Per channel N:

| Register | Offset | Meaning |
| --- | --- | --- |
| OFF count | `N × 0x14` | Low interval in clock cycles |
| CTRL | `N × 0x14 + 8` | Enable mask `0x0b` |
| ON count | `0xb0 + N × 4` | High interval in clock cycles |

Period is OFF+ON. The studied stock backlight uses channel 1 of `0x01000000`,
with a 500000 ns period and duty `level × 40000 + 100000` ns for levels 1–10.
Disabling it writes duty zero while leaving enable set. The buzzer uses channel
0 of `0x01002000`, period 260000 ns and duty `volume × 13000` ns. Volume 10 is
50% duty. These are board policies, not universal PWM hardware limits.

Linux PWM chip numbers depend on probe/allocation order; they are not controller
addresses. Physical brightness and acoustic output are separate from register
or API readback.
