# G11 MCU memory, segmented images and model commits

[Firmware](README.md) · [G11](../devices/skydroid-g11.md)

Physical tests on the investigated G11 established independent 128 KiB flash
and 16 KiB SRAM. These results supersede the initial 64 KiB/8 KiB software
envelope inferred from the package suffix. They apply to the tested unit and
conditions, not every nominal GD32F330C8 device.

## Capacity evidence

The MCU identity reads were:

| Register | Value |
| --- | --- |
| CPUID, `0xe000ed00` | `0x410fc241` |
| Device identification, `0xe0042000` | `0x17040410` |
| Density word, `0x1ffff7e0` | `0x00100080`: reported 128 KiB flash / 16 KiB RAM |
| Direct-boot VTOR, `0xe000ed08` | `0x08000000` |

The density word and an OpenOCD bank declaration alone would not prove usable
capacity. The October 25 SRAM test wrote six patterns—zeros, ones, alternating
bits, addresses and inverse addresses—to all 2,048 words above `0x20002000`.
It checked lower/upper independence and restored both the upper 8 KiB and its
16-byte lower scratch region. No halt, reset or flash operation occurred;
MCU boot identity stayed unchanged. A Linux input-session expiry was timed to
before the RAM writes; explicit reconnect restored input without restarting
Linux or its UI. The test was not a loss-free UART qualification.

The flash test used a separately verified 16,236-byte development image and
RF-off runtime. It froze the watchdog during debug halt, read all 128 KiB and
16 option bytes twice, and saved matching backups. An initial attempt refused
an occupied lower alias-guard page before programming anything. The passing
attempt used blank `0x08009000` as the potential alias of first marker address
`0x08019000`.

After marker write/erase and lower-region checks, the tool programmed an
address-derived pattern and its inverse across all 65,536 upper bytes. It
verified the entire lower flash and option bytes, then restored the upper region
to blank. **All flash and option bytes matched the backup before reset.** The
171.4-second run then restarted the same image, advanced its boot nonce and
restored Linux/Wi-Fi/UI with RF off.

This establishes present read/write independence. It does not qualify retention,
endurance, every page's erase isolation, voltage/temperature margins or upper
instruction-fetch timing. An erased read of `0xffffffff` would not have supplied
this evidence. Uncertain debugger transport was treated differently from a
definite programming failure: no further commands/reset until the potentially
in-flight operation was resolved.

## Fast and stopped-mode regions

The manufacturer reference used by the study describes the first 64 KiB as the
fast instruction region, with long access delay above it. The implementation
therefore keeps live mixer code, IRQ vectors, drivers and active model reads in
lower flash. Upper code runs only during boot or stopped validation/configuration.
Physical upper-fetch deadline qualification remains open.

The first segmented layout kept two 8 KiB model caches at `0x08009000–0x0800d000`,
leaving 36 KiB of fast executable space. The later installed **model-mirror
layout** moves durable caches upward and leaves a disposable active copy below:

| Region | Start | End, exclusive |
| --- | --- | --- |
| Fast executable and initialized-RAM load bytes | `0x08000000` | `0x0800b000` |
| Active model mirror | `0x0800b000` | `0x0800d000` |
| Selected-model journal | `0x0800d000` | `0x0800d800` |
| Preserved device/manifest data | `0x0800d800` | `0x0800f800` |
| Boot nonce journal | `0x0800f800` | `0x08010000` |
| Stopped-mode executable | `0x08010000` | `0x0801b000` |
| Durable model slot 0 | `0x0801b000` | `0x0801d000` |
| Durable model slot 1 | `0x0801d000` | `0x0801f000` |
| Reserved | `0x0801f000` | `0x08020000` |

This provides 44 KiB for each executable region. The mixer binds immutable
model/radio pointers directly to flash; copying the entire configuration into
RAM does not fit the current envelope. Simply moving those active reads to upper
flash would violate the timing design.

The linker captures stopped objects before general text placement and explicitly
groups mutable upstream mixer/curve/GVar/switch data for reset. Constructors and
runtime relocations are rejected. The fast-image-end symbol excludes lower
storage operations; it is not the maximum address of the two-region image.
ARM integration tests trap upper instruction execution **or data reads** while
acquisition runs. Activation must return to fast code before starting inputs,
leaving no upper-flash caller waiting to return afterward.

## Executable installation

The build produces separate lower/upper binaries, each padded to 1 KiB erase
pages, plus an ELF and segment manifest. Verification examines every file-backed
ELF load using physical addresses, including initialized-RAM load data. It
rejects overlaps, absent/cross-region segments, unsafe names, bad padding and
hash mismatches, and reconstructs the bytes against the ELF. Recomputing a
modified binary's hash does not bypass the ELF comparison.

A flat binary spanning both executable regions would also span model, device
and nonce storage. Installation instead programs only declared page ranges,
verifies executable and preserved regions/options, and leaves reset as a
separate verified step. The exact prior ELF and runtime layout matter: old
runtime offsets must not be regenerated from newer headers.

The original default startup used 8 KiB SRAM. The qualified direct-boot profile
uses 16 KiB, stack top `0x20004000` and a 2 KiB reservation. A physical halt
observed SP at `0x20003f78` within that upper stack; the device was reset afterward
rather than resumed with stale ADC timestamps. This is not a worst-case stack
or arbitrary upper-BSS/environmental qualification.

## Disposable active mirror

Mirror preparation requires a fully validated, immutable, journal-selected
durable source and stopped acquisition/control. Exact header/payload equality
permits no-write reuse. Otherwise the implementation erases/verifies eight
mirror pages, copies payload with readback, writes header/commit last, then
compares the full committed result. Only READY publishes a fast model pointer;
even a failed new preparation clears a previous READY state.

The mirror never selects the model. On every boot/activation it must agree with
the durable journal-selected source. Reset during copying recovers the previous
selection; reset after selection commit recovers the new selection. A valid
mirror cannot override a corrupt or mismatched authoritative source. CRC checks
integrity, not authenticity.

Each preparation step performs at most one page erase or program operation,
allowing watchdog service between successful stopped steps. Both source and
destination ownership gates are checked; physical mutation gates recheck stopped
state with interrupts masked. Only durable slots and the exact lower mirror
receive the corresponding write permissions. Running calibration reads the
lower mirror, while upper-slot validation remains stopped-only.

Migration first validates the exact old/new firmware against original/planned
flash snapshots in ARM execution. Both must restore the same selected RF-off
model, working values and test outputs. It stages and verifies **both upper
durable slots before changing lower models or executable regions**, preserving
uncommitted remnants in the inactive slot too. The mirror is prepopulated for
first-boot reuse. Full-flash/option comparisons follow mutation stages.

This is not a power-loss-atomic firmware migration. SWD recovery can be required.
Rollback preserves current valid newer or torn nonce claims to avoid reusing a
boot identity; it refuses regressed/invalid nonce state or changed option bytes.
Ordinary modeled failures and ambiguous transport failures have different
recovery paths. Physical interrupted-write qualification remains open.

## The commit halfword failure

October 26 physical staging exposed behavior missing from the original FMC
emulator. After payload and most header bytes were written to inactive slot 1,
writing final halfword `0x3cc3` at `0x0801d03e` failed with FMC status `0x4`
(PGERR). The last header halfwords were `c33c ffff`. A watchpoint caught the
actual driver return, and a separate halted SWD write to that halfword failed
too; increasing a UART timeout could not fix this programming failure.

Controlled inactive-slot tests, followed by restoration, found:

| Pattern | Observation on this unit |
| --- | --- |
| Header first, ascending halfwords | Passed |
| Payload first, ascending header | Last halfword failed |
| Lower halfword erased, upper already programmed | Passed |
| Upper halfword erased, immediately following halfword programmed | PGERR; target stayed erased |
| Programmed data moved 4/6/8 bytes ahead | Passed |
| Failing adjacency case, value zero | Passed |
| Aligned word `0x3cc3c33c` at `0x0801d03c` with following payload present | Passed |

The fix programs both final commit halfwords together using an aligned 32-bit
callback, after payload verification. It must not first program the low half
and then write the word over it. Format, payload offset and validation remain
unchanged. Partial-word tests still require both markers plus CRC/semantics;
one word operation is not a guarantee of power-loss atomicity.

The deployed fix ELF was
`18c372b2a330cddd3b98e2c190ed74e29a4c286745988b3684fc2f235922581b`.
It then exposed a separate client deadline issue: activation had completed on
the MCU after the client timed out. The uncertain operation was inspected and
explicitly recovered, not blindly replayed. Normal observations retain 150 ms;
stopped SELECT/ACTIVATE received a bounded three-second exchange allowance.

## Physical model qualification

The passing sequence staged an identical RF-off model under a temporary identity,
activated/rebuilt its mirror, returned to the original, restored the original
inactive cache and reused the already matching mirror:

| Operation | Observed elapsed |
| --- | ---: |
| Stage clone | 1,455 ms |
| Activate clone / rebuild mirror | 1,241 ms |
| Select original / rebuild mirror | 1,239 ms |
| Restore inactive cache | 1,450 ms |
| Select original / reuse mirror | 121 ms |

Maximum stopped-management interval was 1,196,737 µs, including work beyond
flash programming. The watchdog survived. Both cached originals, mirror and
334 working-state bytes matched their prior values; only selection journal
bytes changed during the full sequence. Executable/device/reserved/nonce bytes
and options were unchanged. A later ten-second observation bounded control at
499.85–500.23 Hz without MCU/input/flash faults, RF off.

The corresponding simulator covered partial commits and hundreds of mutation
cuts; physical testing established the particular successful sequence and the
real halfword defect. Neither establishes physical power-cut recovery,
worst-case flash/watchdog timing, endurance or radio receiver continuity.
