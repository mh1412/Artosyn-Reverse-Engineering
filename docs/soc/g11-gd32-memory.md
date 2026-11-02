# G11 GD32 identification and memory evidence

[Chip studies](README.md) · [G11](../devices/skydroid-g11.md)

The stock binaries' interrupt/peripheral layout, GPIOB at `0x48000400` and
executed FPU setup identify a GD32F3x0 Cortex-M4F family. The package photograph
was read as GD32F330C8T6; that suffix reading remains unresolved because this
unit's density report and restoring capacity tests indicate 128 KiB flash and
16 KiB SRAM.

| Register | Observed value |
| --- | --- |
| CPUID, `0xe000ed00` | `0x410fc241` |
| Device identification, `0xe0042000` | `0x17040410` |
| Density word, `0x1ffff7e0` | `0x00100080`: reported 128 KiB flash / 16 KiB RAM |

## Capacity measurement

The October 25 SRAM experiment tested all 2,048 words above `0x20002000`
with zero, one, alternating-bit, address and inverse-address patterns. It
checked lower/upper independence and restored the upper 8 KiB and the lower
scratch region. No halt, reset or flash write was part of that SRAM test.

A separate flash experiment backed up all 128 KiB and option bytes twice,
checked potential lower-address aliasing, then wrote an address-derived pattern
and its inverse across the upper 64 KiB. It verified lower flash and option
bytes and restored the upper region. All flash and option bytes matched the
pre-test backup before reset.

These were dedicated hardware-capacity experiments, not observations of normal
stock application execution. They establish usable independent memory on the
tested unit, not retention, endurance, voltage/temperature margins or upper-flash
instruction timing. A density word or erased read alone would not prove this.

The [stock MCU reconstruction](../firmware/g11-stock-mcu-reconstruction.md)
records the installed 0.5 image, official 0.8 download and their distinct hashes.
The [battery study](g11-battery-and-service-inputs.md) retains stock ADC conversion
and service-input findings.
