# GD32 flight-controller programming references

[References](README.md) · [Documentation index](../../README.md)

The adjacent flight-controller work supplies GD32H757VI and GD32F460RG
probe-rs targets. These are separate MCU families and boards from the
[G11 GD32 acquisition study](../soc/g11-gd32-acquisition.md). Their geometry and
flash-controller registers must not be transferred to G11 merely because the
manufacturer is the same.

## Recorded target profiles

| Target | Recorded board | Flash geometry | Algorithm |
| --- | --- | --- | --- |
| GD32H757VI | GD32H757V1STARTV1 | 1 MiB at `0x08000000`, 256 uniform 4 KiB sectors, 32-bit programming | Custom FMC implementation |
| GD32F460RG | HAKRCF460V2 | 1 MiB at `0x08000000`; four 16 KiB, one 64 KiB and seven 128 KiB sectors | Vendor F4 algorithm retained from probe-rs |

The H757 research notes report FMC at `0x52002000`, rather than the
`0x51102000` SDK definition used for comparison. They also report a read fault
at the first address beyond the observed flash, `0x08100000`, contradicting the
2 MiB Betaflight target assumption used in that investigation. These are
board-scoped recorded findings, not freshly repeated hardware measurements.

The F460 profile records 64 KiB TCM at `0x10000000` and 448 KiB main SRAM at
`0x20000000`. Its baseline was the GD32F470VG profile from probe-rs v0.32.0,
commit `48f5e4d53c690a1d40c2454033c6f785b4f4f95c`, with the unchanged
`gd32f4xx_1mb` algorithm originating in GigaDevice DFP 3.0.3. That reuse is
supported by the recorded geometry and board test, not by a general assertion
that F460 and F470 are interchangeable.

The README reports a verified Betaflight HEX download and reset on HAKRCF460V2,
with the configuration sector at `0x08004000` remaining byte-identical. Its
selective download omits chip erase. A chip erase includes that configuration
sector and has a different preservation boundary. This review read the report
and H757 algorithm; it did not independently reproduce those flash operations.

## Custom H757 algorithm behavior

The algorithm exposes init, uninit, sector erase, page program and mass erase
entry points. Its FMC register offsets are KEY `+0x04`, CTL `+0x0c`, STAT
`+0x10` and ADDR `+0x14`. Initialization writes unlock keys `0x45670123` and
`0xcdef89ab` when the lock bit is set; uninit sets the lock bit again.

The completion loop polls STAT busy bit 0 and reports failure for bits 17 or
18. It has no internal polling timeout, and it does not clear those error bits.
Initialization returns success without checking that the unlock succeeded.
These limits distinguish the small target-resident routine from the host
programmer's supervision and verification.

Programming writes one 32-bit word at a time, padding a final partial word
with `0xff`, with instruction/data barriers around the operation. The source
does not perform its own address-range/alignment validation or readback
comparison. Correct geometry, caller constraints and separate verification
remain part of the programming procedure. The historical notes about missing
tool support describe the development context; current upstream support was
not surveyed here.
