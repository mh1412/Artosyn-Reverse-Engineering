# AR8030 command construction and physical frame duration

[Firmware index](README.md) · [Automatic channel scanning](../protocols/ar8030-automatic-channel-scan.md)

This study covers the archived experimental stage-162 branch for Gnd485
18.21.10. It reconstructs command construction and duration from the original
runtime loaded at `0x204000`, SHA256
`3e454131a334ec422c0b2e8ae4924d6bda68cb61b70fd754da26412578b39c87`.
The comparisons use original instructions with synthetic state and intercepted
helpers. They establish specific software contracts, not physical command-engine
timing or the capabilities of every later Open8030 release.

## Descriptors, physical slots and logical metadata

The builder at `0x21dfea` expands each descriptor into a preparation slot and a
body slot, applies role conversion, writes command/user fields and the stop
marker, clears the pre-command bank, then installs enabled pre-commands. Their
displacements use signed wrapped arithmetic. Receive conversion clears the
descriptor's pre-command enable.

The native builder validates before writing. It rejects raw user 15 as an input
descriptor and accepts at most 47 descriptors, leaving room for termination in
the 96-slot physical table. Unknown opcodes retain their observed expansion
behavior. Those limits differ from the separately retained logical list:

| Representation | Contents and limits |
| --- | --- |
| Input descriptor | User, opcode/mode and optional pre-command information |
| Physical command bank | Expanded prepare/body slots, user nibbles and stop bit; 96 slots |
| Logical metadata | Up to 20 selected body entries; raw user, direction, bandwidth, opcode, band, RX flag and frequency |

Rebuilding resets logical count and cursor to `-1`, while preserving untouched
metadata fields and the separate scan channel. Unknown mapped logical user 12
gets fallback band/bandwidth while retaining frequency and flag. These native
structures are implementation-owned representations, not recovered ABI structs.

Updater `0x2212f6` changes the first matching entry. Logical users 8/9 map by role
and direction; logical users 10/11 map to raw users 12/13. Most selectors match
both raw user and direction. A mode update matches raw user alone, and RX-flag
updates leave TX entries unchanged. Unrelated fields survive.

The archived comparison covers 568 builds, 179,867 ordered writes and 6,720
updates, including 780 updates that changed metadata. It executes original
builder/updater/user-map instructions; allocation, interrupt locks, logging,
field writes and low-level profile accessors are intercepted. Nonzero scratch
seeds check that matching results do not depend on zeroed allocation.

## DEV mode-0 frame composition

The initial integrated continuous operation installs three logical descriptors
while TX and RX are held: broadcast RX, independent power measurement and slot-0
TX. Tuning and PHY profile changes update the corresponding metadata. Copied
command snapshots describe installed software state; they do not observe the
hardware's current slot.

The later producer reconstructs the mode-0 portion of `0x22c774`. It uses
broadcast user 8, raw user 13 and slot 0, with profile pre-command modes masked to
three bits and displacements sign-extended from eight bits. Enabled band-auto
with suppression clear adds two descriptors:

| Descriptor | Purpose |
| --- | --- |
| `0x325` | Raw user 12, opcode 5: automatic scan |
| `0x39c` | Raw user 14, opcode 12: wait |

Before installing that frame, it writes `BB+0x988[15:0] = 15000` and
`BB+0x98c[31:28] = 0`. Acquisition suppresses the additions, matching caller
`0x21da4c`; the LINK `0x200` transition passes suppression zero, matching
`0x22caac`. Early producer checkpoints still required manual loader policy, so
their standalone five-descriptor tests did not mean the normal runtime used it.
The later automatic-loader checkpoint connects that policy and lifecycle.

Original descriptor bits 14/15 contain stack residue; native clears them. The
builder ignores these bits, so the comparison checks final writes and decoded
metadata rather than equating padding. The retained producer comparison covers
496 cases, 296 automatic frames and 164,048 writes. Modes 1/3 and their wider
slot-mask/duration policy remain outside this producer.

## Duration of the installed physical frame

The frame-duration follow-up replaces the old three-command sum with a reader of
the installed physical bank. It stops **before** the first stop-bit slot, or
after 96 slots. Preparation slots contribute time: three logical descriptors
expand to six physical slots, and the automatic frame has ten.

The read callback requires a stable register image. The native helper performs
no writes, allocations, profile reconstruction or timer changes. For each slot,
it calculates `(count × unit_ns + 500) / 1000` with integer division, then sums
the rounded microseconds. Rounding only after summation gives different results.

All offsets below are relative to BB. Mode is the physical command's two-bit
mode, and low/high refer to 16-bit halves unless specified otherwise.

| Selection | Count field | Unit |
| --- | --- | --- |
| Raw users 0–7, mode 2 | `0xa00 + raw*2`, halfword access | 640 ns |
| Raw users 0–7, other modes | `0xa10 + raw*2`, halfword access | 640 ns |
| Raw users 8–9 | `0x970`, low for mode 2, high otherwise | 640 ns |
| Raw users 10–11 | `0x99c`, low for mode 2, high otherwise | 640 ns |
| Raw user 12 / 13 | `0x980`, high / low | 640 ns |
| Opcode 6 / 7 | `0x97c`, high / low | 640 ns |
| Opcode 9 / 10 | `0x984`, low | 640 ns |
| Opcode 12, raw 14/15 | `0x988` low plus `0x98c[31:28]` as bits 19:16 | 20 ns |
| Opcode 13, raw 14/15 | `0x988` high plus `0x98c[27:24]` as bits 19:16 | 20 ns |

Opcodes 6/7/9/10 force raw user 15 and ignore their user-table nibble. Otherwise
raw-user routing takes precedence over opcode-specific fallback. Unmatched
combinations contribute zero. A null reader returns zero.

One synthetic fixture uses prepare count 7, RX8 count 8330, user13 count 156,
TX0 count 3491, user12 count 1053 and wait count 15000. Original results are
7,677 µs for six slots and 8,659 µs for ten. The 982 µs addition consists of two
4 µs preparations, 674 µs for user12 and 300 µs waiting. These are calculated
fixture results, not measured airtime.

## Cached timing and policy-only rebuilds

Original caller `0x22caac` stores duration at context `+532`, byte interval
`(duration + 500) / 1000` at `+148`, and dwell `(duration + 2000) / 1000` at
`+136`. Addition wraps at 32 bits before division; the interval then narrows to
a byte. Fifteen original caller cases compare these writes and the ordered
frame-build, duration-read and monitor-init calls, with those helpers intercepted.
Native startup accepts rounded timer intervals only in `1..255` ms.

Subtype-9 policy application calls the frame producer directly, without that
startup caller. Consequently the native policy-only rebuild preserves cached
duration, monitor history and pending hop state. Recalculating every timer on
every policy change would not follow this observed call boundary. See the
[channel integration study](../protocols/ar8030-channel-integration.md) for DMA
deferral and held-engine ownership during those changes.

## Evidence and remaining limits

The retained original duration run has 1,809 successful cases, one allocation
failure returning zero, and 6,382 prefix comparisons. It executes table counting,
decoding and summation on a stable synthetic BB image. Field reads, allocation,
free and trailing memset are intercepted. Profile reconstruction with flags 2,
concurrent table mutation and actual RF timing are outside its scope. The partial
processor model's added packing instruction was checked with three synthetic
fixtures; this was not a physical instruction test.

During this wiki review, freshly compiled native duration code matched all 1,809
successful totals and 6,382 prefixes. The 15 timer-cache records, original image
mappings, four instruction ranges, 451 retained disassembly locations and both
dispatch-table hashes were checked again. This replay reused the recorded
original results; it did not rerun the original emulator or hardware.

The archived follow-up reports passing RF/adapter sanitizers, 42 Python tests and
four deliberately failing negative controls for wrong wait units, truncating
rounding, ignoring the stop bit and retaining the old duration helper. A shared
reply encoder reduced code duplication; 16,388 length/status/alias comparisons
and reconstruction of 26 replaced response blocks support preserved wire
behavior. Configuration packing retained the 436-byte channel-plan capacity.
These wider suites were not rerun for the wiki.

The final retained runtime is 94,068 bytes, SHA256
`d3bf5db438fdffc4b01008d8cc98283014804336bcf6aa35e80f79a45ac62e5c`;
its bytes were verified during this review. Allocated RAM is **704,512 of
704,512 bytes**, with no headroom in the build accounting. Preserved socket,
payload, history and stack capacities do not establish runtime stack safety.
Hardware trial 162 had not run at that offline checkpoint. Its
[subsequent physical test](../experiments/open8030-baseline-revalidation.md)
used this exact runtime and failed acquisition, delivering no video. The next
trial restored video using the older stage-145 runtime. Normal level-11
acquisition, wider RF policy and full replacement acceptance remained unresolved.
