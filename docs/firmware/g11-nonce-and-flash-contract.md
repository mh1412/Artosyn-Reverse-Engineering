# G11 boot nonces and flash ownership

[Firmware](README.md) · [Current memory layout](g11-mcu-memory-and-models.md) · [Update history](g11-mcu-update-history.md)

## Persistent boot identity

Custom sessions need a never-reused boot identity so a recorded old Linux
response cannot authorize output after MCU reset. The allocator uses two
1 KiB pages. Each has a 32-byte header and 496 halfword slot markers. The header
contains magic, a 64-bit epoch and complement, capacity and complement,
reserved words and a final commit marker.

```text
nonce = epoch × 496 + slot + 1
```

Programming and verifying a zero marker consumes the nonce before publication.
A partially programmed marker also consumes that slot after restart; used
slots must be contiguous. When full, the allocator erases the other page,
commits the next epoch header and consumes its first slot. It never erases
the newest committed page during rollover. Two committed headers must have
consecutive epochs. Unknown storage with used slots and no valid header fails
closed; interrupted first provisioning before slot use may restart.

Failures leave the caller's output unchanged, though a failed allocation may
consume an identity. Exhaustion refuses further allocation. The installed
layouts place the pages at `0x0800f800` and `0x0800fc00`; early notes saying no
addresses were selected describe the portable prototype. Updates must preserve
these pages. Restoring an old full-flash snapshot can roll back the nonce and
is outside the allocator's uniqueness guarantee.

The model assumes verified reads, exclusive ownership, programming that only
clears bits in the addressed halfword, and erasing that only sets bits within
the addressed page. Operations may partially or fully apply before failure.
Arbitrary corruption, malicious rollback, out-of-spec wear and voltage faults
violating these assumptions are not covered.

Portable tests exercised 8,368 interruptions across provisioning/allocation/
rollover, four partial-bit patterns, both erase orders, all 65,536 torn slot
values, unknown storage, duplicate epochs and exhaustion. Those are model
properties, not exhaustive analog flash or physical power-cut qualification.

## FMC adapter contract

The initial counter/serial-update binding used the nominal lower-64-KiB envelope;
later model profiles expanded placement. In every profile the linked image end
includes all flash sections and RAM initializers, not just executable text.
Caller layout must separately exclude device data; a linker gap is not proof
an address is unused.

One zero-initialized state owns FMC, shared by its allowed clients. Binding
does not unlock or mutate flash, and invalid binding leaves output callbacks
unchanged. Operations require an idle, locked controller, no conflicting
control mode and bit programming disabled. An unlocked/busy controller is
not taken over. Unexpected ownership/readback/state faults latch the driver
off until reset. Bad caller indices/addresses fail without a hardware mutation.

Normal interrupts are masked for a transaction while preserving prior PRIMASK.
NMI/hard-fault handlers must not access FMC. The adapter emits standard unlock,
W1C flags, halfword program, single-page erase and relock operations; it has
no mass-erase or option-byte operation. Programming requires an erased halfword;
writing `ffff` is a verified no-op. Completion requires BUSY clear, ENDF set,
no PGERR/WPERR, controller relocked and exact readback. Erase verifies all
512 halfwords, including the final one.

Polling budgets count register reads, not elapsed time. Executing from flash
can stall during erase/program; this is not a claim of uninterrupted UART/RC
service. After a BUSY timeout the driver does not alter control bits while
an operation may still run. Board reset/watchdog recovery must establish a
usable controller again. The [resident erase-pacing defect](g11-mcu-update-history.md#physical-erase-pacing-correction)
demonstrated why that distinction matters physically.

## Region-specific clients

The historical resident updater bound an explicit application region and
separate manifest above the loader, excluding nonce/vendor/device/unallocated
pages. Reads were 1–128 bytes wholly inside one permitted region, programming
was aligned halfwords, and erase was aligned 1 KiB pages. Range checks preceded
MMIO/flash access. A shared FMC fault disabled both updater and nonce clients.
Those callbacks were not arbitrary-address access.

The initial SWD configuration-cache profile reserved 36 KiB code with 8 KiB
slots at `0x08009000` and `0x0800b000`; it was a staging layout, superseded by
the [later mirror/durable model map](g11-mcu-memory-and-models.md). At that stage
reservation alone did not imply an installed model store or autonomous mixer.

Configuration binding used slot-index/offset callbacks, excluded code/device/
nonce ranges, and refused coexistence with an application-updater binding.
Active-slot protection belonged to the higher-level configuration store.
Each mutation rechecked the caller's quiescence gate with IRQs masked immediately
before FMC access. The gate had to ensure RF was inhibited and no time-critical
control needed service throughout the operation. A normally false gate paused
work; revocation between the outer check and final adapter check faulted the
store without mutation. This was stopped-only flash work, not an in-flight
background persistence service.

ARM/FMC-model tests included 1,100 nonce claims and 105 interrupted-operation
cases, 825 update cuts, and 477 cache cuts across 159 mutations. They checked
range preservation, final gates, readback, ownership and shared fault latching.
Semantic validation in early cache fixtures was only a test rule. Later
physical model work exposed a halfword-adjacency PGERR missed by those models;
the aligned commit-word fix and measured activation times are documented in
[memory and model commits](g11-mcu-memory-and-models.md). Passing emulation
therefore must not be generalized into complete hardware programming behavior.
