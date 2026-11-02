# AR8030 DEV receive startup and search

[SoC studies](README.md) · [AR8030](ar8030.md)

This study reconstructs stock Gnd485 18.21.10 receive startup from instruction
analysis and bounded diagnostic observations. Private diagnostic operation
numbers are experiment selectors, not vendor host-command numbers.

Register offsets are relative to baseband base `0x0a110000`. Function addresses
refer to the original runtime loaded at `0x204000`, SHA-256
`3e454131a334ec422c0b2e8ae4924d6bda68cb61b70fd754da26412578b39c87`.
Synthetic disassembler symbols and custom-instruction models are analysis aids,
not original vendor declarations.

## Search and maintenance are different timers

The DEV node has two relevant callbacks:

| Callback | Link-task message | Role |
| --- | --- | --- |
| `0x2287e4` | `0x205` | Advance search channel, retune, reset/release and rearm |
| `0x228918` | `0x206` | FCH monitoring/link maintenance through `0x2362ec(0)` |

The AP hopping branch in `0x2362ec` requires role zero. It does not describe
the DEV role-one startup search. DEV maintenance uses `0x230f74` and node field
`+0x98`; the loss callback at `0x22d7ba` returns while broadcast user 8 is in
state zero. Treating that callback as the initial acquisition timer produced
the wrong reconstruction.

Startup message `0x200` builds the frame through `0x22caac`, tunes and prepares
RX user 8 through `0x22aaea`, sets the broadcast buffer mode, pulses the paired
reset/release helper at `0x22aa8c`, and sets search flag `0x10`.
Message `0x205` reaches `0x22f28e`, which requires that flag. Channel selection
uses `0x22931c -> 0x2284aa`, checking entry bit 24; selector 3 admits both band
classes. Wrapping through `0x229312` resets the index and can select another
bank. `0x228902(node, 1, channel)` selects RX8 and restarts the search.

Frame construction at `0x22cbbe` maintains duration at node `+0x214`, maintenance
interval at `+0x94` and search dwell at `+0x88`. Ordinary command durations round
each register count multiplied by `640 / 1000` to microseconds **before** adding
the durations. Dwell
alternates between `floor(F_us / 1000) + 2` and
`2 × (floor(F_us / 1000) + 1)` milliseconds at list wrap.

The initial six-command profile produced `F_us = 9427`, hence 11/20 ms dwell.
The configured 5 MHz profile later produced `7096`, hence 9/16 ms. Diagnostic
operation 36 reconstructed a bounded single-channel retry; operation 39 added
the supplied channel list. Delayed polling performs one retry rather than a
burst of missed retries. A reset timeout halts the experiment with `-110`.
The early diagnostic stopped retries on a validity indication, receive event
or pending DMA. That latch was not the stock link state machine.

## Identity, held engines and interrupt observations

Acquisition helper `0x222cac(8, 0)` writes the low byte of **both** `DE0` and
`DF0`. Its `user & ~8` branch shares the relevant handling for users 0 and 8.
An earlier reconstruction updated only `DE0`. The one-byte identity is not
the full four-byte peer address; the physical meaning of every packed identity
field remains unresolved.

Mode-zero startup calls `0x21db2e` to set `800` bit 2 while RX is held.
RX preparation releases bit 1; the paired reset helper pulses bits 0/3 while
preserving bit 2. Earlier native PSRAM preparation had cleared bit 2 and never
restored it. Operation 37 corrected that hold, but produced no receive packets.
Later link code can release the transmit-side engine; doing so requires its
MAC/TX context, not just a register write.

The stock dispatcher at `0x222d50` uses 20 entries at `0x2541ac`. Sources 0–3
map to CLIC vectors 72–75, sources 4–19 to vectors 80–95. It pulses the source
bit at `AF4`, clears CLIC pending state, then runs pre-hook, handler and post-hook.
The native poller followed that order. This does not reproduce the entire
interrupt-entry ISA or RTOS scheduling behavior.

| Source | Observed handler behavior | Interpretation limit |
| --- | --- | --- |
| 2 | `0x228066` samples `E6C`; optional callback registry at `0x249044` | Activity is not a decoded header or received packet |
| 5 | `0x21f804` byte-swaps `E70` and posts PHY message `0x100` | Raw changing power values have no established calibrated units |
| 12 | End-of-packet observation; user comes from `FBC` bits 20–23 | Distinct from interleaver source 18 |
| 14 | `0x24531c` temporarily clears `BB0` bit 7, reads `F00` bit 7, restores selector | A hardware validity flag alone does not validate control contents |
| 15 | `0x21eb60` reads `F48` and logs/calls an optional callback | No additional hidden receive-release gate was established |
| 18 | Interleaver handling at `0x227d58` | Must not be relabeled as end-of-packet source 12 |

Source 2's special SNR-message condition is `(raw_user & 0x0d) == 0x0c`, which
does not hold for DEV broadcast user 8. The common registry has eight optional
entries; its existence does not establish which callbacks were registered in
a particular trial. Polled `E6C` attribution is not an edge-latched snapshot.

## Configured startup differs from a later linked profile

The external configuration selected a 5 MHz broadcast receive profile with
payload interleaving length 12, count 2. An earlier linked snapshot used 20 MHz
and length 24, count 2. They are different operating states.

The parser at `0x23253e` maps lengths 3, 6, 12, 24 and 48 to codes 0–4 using
the table at `0x253cc4`. Loading user 8's broadcast profile passes through
`0x234a78`, `0x245352` and `0x249106`; the PHY path subsequently applies the
profile to its users. RX bandwidth helpers also update software metadata, so
an audit limited to new MMIO stores misses part of the operation.

The interleaver mapper combines the enabled-payload setting, length code 2 and
count code 1 into `9E0[2:0] = 4` and `9E0[15:13] = 5`. These packed fields are
not simply copies of the configuration codes. Complete observed words retain
the accompanying fields without inventing meanings for unexplained bits:

| State | `9E0` | Duration/dwell |
| --- | --- | --- |
| Configured DEV startup | `3cb0a2ac` | 7,096 µs frame; 9/16 ms search dwell |
| Earlier native configured profile before upper-byte correction | `20b0a2ac` | Same intended configured timing |
| Diagnostic linked-profile hypothesis | `3cb0c4ad` | 9,427 µs frame; 11/20 ms dwell |
| Normal-stack linked observation | `20b0c4ad` | Separate later 20 MHz state |

The startup tuning path at `0x24c93c` supplies upper byte `0x3c`; a state-one
link-profile path through `0x2371e4` supplies `0x20`. Their precise physical
meaning is unresolved. Neither value alone defines a complete acquisition or
linked state.

The recovered duration formula gives 3,000 µs for length 12, stored as count
4,688 (`0x1250`). Length 24 gives approximately 5,331 µs, count 8,330 (`0x208a`).
Changing just bandwidth left the old frame duration; operation 42 changed the
interleaver duration too. Neither change by itself yielded packets in the
1.5-second trials.

## Bounded channel and power experiments

Operations 39–42 accepted four LE32 words—initial frequency in kHz, duration
in ms, zero-padded identity and operation—followed by 1–42 frequencies.
The first list entry had to match the initial frequency. The implementation
copied and validated the plan, including extent, duplicates and supported PLL
ranges, before MMIO. It held/reset RX before retuning and acknowledged stale
power/command status while held. A failed hold did not advance the PLL or visit
counter. This was stronger isolation than stock's live retune, not identical
timing.

Operation 38 used the stock power-search command description, including special
users 13/14, while keeping TX held and clearing prior transmit metadata.
Operation 40 repeated that power diagnostic across the supplied list every
10 ms; it did not implement stock's averaging policy. Power peaks and complete
channel-list visits establish neither peer identity nor a transmitter hop order.

## Prestart snapshots and proposed register corrections

Trial 67 instrumented stock with a two-byte return at `0x22aaea`, skipping its
buffer/reset-release/timer setup. Earlier preparation could still execute.
Two 4 KiB snapshots 100 ms apart differed only at timing register `E4`, but
that stability does not make live pointers, unused banks or skipped-helper
state suitable initialization constants. Stock was on 4.915 GHz; the main
native comparison was on 5.770 GHz. `BF0 = 41410101` was observed before the
skipped helper, not proof that native startup must begin with that value.

The compensation defaults included `480 = 00400c04`,
`48C = 08040000`, `4A0 = 00ff0000`, `4A4 = 0` and `4A8 = ff`.
Operation 44 restored only `480` bit 1, `4A0` low 16 bits, `4A4` high 16 bits
and `48C` low byte, preserving neighbors. It did not port the entire conditional
command scheduler at `0x223128`; that code depends on metadata and last band.
Generated pseudocode had also misinterpreted a nonreturning helper boundary.

Initializer `0x22373a -> 0x2236e0` programs the `B4C` user field with encoded
value 1 for DEV and 17 for AP; the observed DEV word was 8. Operation 45 restored
that field. Its physical meaning remains unknown. Later handler `0x227d58`
updates it from the decoded user, so it is not an unconditional static constant.

The proposed `A48` correction also has state dependence. Command handler
`0x223644` writes `40404040` for next-TX bandwidth code 4, `80808080` for code 5,
and zero otherwise. Operation 49 tested zero for its native TX codes 1/2.
Working-stock startup nevertheless retained `80808080` before link acquisition;
the eventual ten-second native trial used that startup value.

## FCH bank reads and checksum gates

The seven-byte control reader at `0x22d30a` sets `BB0` bit 7, samples
`F04/F08/F0C/F10` and `F34/F38/F3C`, then restores the selector. This is the
opposite selector operation from the source-14 validity helper. Packing skips
the first 28 bits of the first group. Raw nonzero register words therefore do
not imply that the extracted seven control bytes are nonzero.

The check at `0x2299d2` forces the final byte's high nibble to `0xf`, starts the
lookup accumulator at zero, and processes bytes 1–6. Stored byte 0 is compared
with that result; a mode-specific path compares only the low nibble. Seven zero
bytes compute `0x1e` and fail both comparisons. The table at `0x253668` is the
[release-specific lookup](../protocols/ar8030-wireless-format.md#release-specific-lookup-checksum),
not an interchangeable standard CRC-8.

Stock checks contents before peer acceptance and cancels search only on its
accepted transition. Early native code latched on any hardware-valid indication,
so it could turn one invalid sample into an apparent “one header per run” limit.
Operation 51 retried all-zero controls but stopped at a nonzero invalid control;
operation 52 retried every checksum failure. Neither operation independently
established valid MAC reception.

FCH diagnostics, DMA integrity, full-control decoding, peer admission and video
delivery remain separate evidence stages.
