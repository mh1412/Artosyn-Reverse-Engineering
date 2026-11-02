# AR9311 slice and incomplete-picture decoding

[Experiments](README.md) · [AR9311](../soc/ar9311.md)

Scope: Gnd485 AR9311 / Wave521C media experiments consolidated on 2026-10-26,
using the original Artosyn codec firmware and a custom Linux host driver.

## Question

Can the decoder expose correct regions of a picture before all its slices arrive,
then accept the remaining slices and finish that same picture? Such behavior
could support lower latency, but decoding, region completion and display safety
must be established separately.

## Setup and controls

The investigation used H.264 and H.265 pictures with controlled slice boundaries,
known visual patterns and host-decoded reference pixels. Progressive-input tests
withheld later data while observing decoder output. Separate prefix tests supplied
one, two or three of four slices and explicitly finalized the incomplete picture.

Firmware/parser probes and temporary driver changes were part of the investigation;
these results do not describe an unmodified general-purpose decoder API.

### Delayed-slice memory experiment

The controlled 640×384 I420 stream contains three pictures with four 96-row
bands. Y/U/V colors are `(81,90,240)`, `(145,54,34)`, `(41,240,110)` and
`(210,16,146)`, rotated between pictures. Independent HEVC slices begin at CTU
addresses 0/60/120/180 with CTU size 32; H.264 slices begin at macroblock
addresses 0/240/480/720. Deblocking and applicable cross-slice filtering/SAO
are disabled, with no B pictures.

Each slice is followed by a legal filler NAL containing 2,048 `ff` bytes and an
RBSP stop bit. This provides parser lookahead without delivering the withheld
slice. References decode those actual padded streams rather than assuming raw
source colors survive encoding exactly.

Frame 0 arrives at 0 ms; target slices arrive at 1,000/1,500/2,000/3,000 ms;
frame 2 arrives at 4,000 ms. Test end mask 49 marks packets 0/4/5 as complete
pictures. Memory snapshots are taken before submitting each target slice, with
an additional delayed snapshot before the final slice and a completed-target
positive control.

The observations cover linear pixels, compressed/FBC allocation, Y/C tables and
motion-vector storage. Allocations are seeded with `0xa5` before ownership
transfer. Coherent/write-combined mappings retain normal allocation ownership;
the snapshots do not create a device completion fence and could tear during
concurrent writes. Each full snapshot costs roughly 43–50 ms, so this is not a
pipeline latency benchmark.

The four initial runs produced 540 snapshot records. Every pre-final sample was
identical to its before-target baseline; completed pictures matched all new
bands and changed compressed/table/MV storage. Internal SRAM-only activity was
not observed by this method.

### Driver and output-layout controls

The first matrix ran five configurations on each codec:

| Configuration | Reorder disabled | Read-pointer-valid on new decode command | Picture-end mask |
| --- | ---: | ---: | ---: |
| Baseline | No | No | 49 |
| No reorder | Yes | No | 49 |
| Explicit read pointer | No | Yes | 49 |
| Both | Yes | Yes | 49 |
| Open target boundary | No | No | 1 |

Mask 1 marks only frame 0 complete, leaving target release dependent on following
VCL and final drain on EOS. Read-pointer-valid is not asserted during an in-flight
refill. Disabling reorder sets `FORCE_FB_LATENCY_PLUS1=1`. All ten runs produced
exact references without early sampled memory changes; opening the boundary
delayed completed-picture release rather than exposing partial output.

A second ten-run matrix compared planar I420 and NV12, bandwidth optimization,
and their combination with reorder/read-pointer controls. Capability word
`0x08818003` has the relevant support bit 15 set. Bandwidth optimization uses
`W5_SFB_OPTION` bit 26; it controls selected compressed/linear writes rather
than declaring a slice-output mode. NV12 observations were converted losslessly
to I420 for reference comparison. All 30 completed frames in that matrix were
exact, with no pre-final linear/FBC/table/MV changes.

Secondary AXI remained disabled because Wave521 SRAM sizing/accounting was not
implemented and this integration had known corruption with it enabled. CODA980
row/field controls and encoder-side SFS names do not establish equivalent
Wave521C decoder controls. Untested firmware/modes remain outside the result.

## Results

### Why the exported low-delay API does not qualify Wave521C

The preceding source/binary audit examined `libmpp_service.so` from the live
17.5.3-labelled system and the extracted 17.5.8 image. Both are **the same**
4,394,048-byte binary, SHA-256
`e480bdc292ede1ca19e24998c7323fbc0218fffe1f8f5d569f1fae384d2f715f`.
They are one implementation, not independent confirmation across versions.

The library exports `VPU_DecGiveCommand` and `GetLowDelayOutput` alongside both
Coda9 and Wave5 backends. The retained disassembly explains why exporting those
names is insufficient:

| Check | Observation in the matched AArch64 library |
| --- | --- |
| Backend dispatch at `ProductVpuDecode`, VA `0xd7f50` | Instance product field at `+20`: IDs 0/1 select Coda9, 2–4 select Wave5; other values reject |
| Low-delay configuration | At `0xe19f0`, load product field `+20`; `0xe19f4` rejects nonzero IDs |
| Low-delay output request | Checks that low-delay is enabled, codec mode is zero and the instance matches the pending instance before the call at `0xe2bcc` |
| Called implementation | `GetLowDelayOutput` is at `0xe6b78`; `0xe2bcc` is its caller, not its entry point |

The related StarFive SDK snapshot at commit
`b60da16be1b36453aa46599889c28310fd148fb8` restricts `SET_LOW_DELAY_CONFIG` to
CODA980 and describes H.264 macroblock-row completion with reordering,
tiled-to-linear and post-rotation disabled. That is consistent with the legacy
backend gate; its register semantics cannot be transferred to Wave5 merely
because some offsets overlap.

The inspected Linux 6.18.36 Wave5 path uses ordinary picture decoding and exposes
picture-complete/bitstream-empty events, without a named decoder row-ready event.
Bitstream-empty reports starvation, not completed pixels. Register offset
`0x300`, `W5_ENC_PIC_SUB_FRAME_SYNC_IF`, and the related SDK's SFS configuration
commands are **encoder-side** interfaces.

The Artosyn Wave5 command path shifts the 32-bit option-structure field at `+12`
left by **5**, and the field at `+16` left by **6**, then ORs both with its selected
base option into register offset `0x104`. The earlier order in this page was
reversed. These operations do not mask the fields to one bit: a value of 2 at
`+12` also contributes `0x40`. Their exact matching structure definitions were
not recovered. They are not identified slice switches, and differently laid-out
SDK structures cannot assign their meanings by analogy. This audit identifies
limits of the examined interfaces, not a silicon-wide impossibility result.

### Complete low-delay helper and Wave5 submission review

The full retained functions were checked against the matching library's ELF
symbol extents and instruction bytes: `GetLowDelayOutput` contains 491 AArch64
instructions at `0xe6b78–0xe7324`, and `Wave5VpuDecode` contains 265 at
`0xee458–0xee87c` (exclusive ends). Their separately retained 17.5.8 listings
have identical instruction streams. This strengthens the identity check; it
does not create a second implementation or a new hardware result.

`GetLowDelayOutput` reads the legacy register bank and assembles a result from
those values and saved instance state. Examples include:

| Register offset / operation | Visible effect in this helper |
| --- | --- |
| Reads `0x1c4`, `0x1dc` | Store result words at offsets `+0`, `+8`; unsigned comparisons against 30 gate indexed metadata paths |
| Read `0x1bc` | Split upper/lower 16 bits into result words at `+112`, `+116` |
| Conditional reads `0x1e0`, `0x1e4` | Split offsets and subtract them from dimensions; coordinate interpretation follows this arithmetic |
| Read `0x1cc` | Extract picture/status bitfields, including bits 18 and 21 |
| Conditional `vdi_read_memory` | Read eight bytes for auxiliary metadata; its return value is not tested |
| `memcpy` calls | Copy 128-byte instance-state blocks into the result, not decoded picture planes |

The complete helper's only call targets are register reads, memory reads and
`memcpy`. It contains no decode-command issue, register-write or wait call. Its
single normal return sets `w0=1`, including paths where the index comparisons
skip metadata lookup. That value is not itself proof of a valid frame index,
fresh pixels, a completed row or an independently checked hardware result.
Result fields are written conditionally; the helper does not first clear the
entire output object. The caller's low-delay/product/codec/pending-instance
gates remain essential, and overlapping register numbers must not be mapped
onto Wave5 meanings by name alone.

`Wave5VpuDecode` follows a different submission path. It prepares register
options, issues command `0x100` through `Wave5BitIssueCommand`, then calls
`vdi_wait_vpu_busy` for register offset `0x70`. A wait result of −1 returns
numeric code 16, optionally logging first. Otherwise it reads `0x1e0`, saves
the low 16 and next 8 bits in instance state, and checks register `0x108`.
A nonzero value returns zero from this function. This is command-path acceptance;
there is no decoded-row wait or output-picture retrieval in this function.

When `0x108` is zero, the function reads failure reason `0x10c`, dumps diagnostic
information and selects these numeric returns. The comparisons are exact
equality tests, not a general bitmask decoder.

| Failure reason | Function return |
| --- | ---: |
| `0x1` | 25, after also reading `0x110` for logging |
| `0x40` | 15 |
| `0x20000` | 16 |
| `0x10000` | 28 |
| Other values | 1 |

An unsupported value in the decoder-state block's mode field at `+76` takes an
earlier return of 3 before the submission register writes. The block is reached
through the instance pointer at `+32`. These numeric mappings
describe this binary; they do not substitute for its unrecovered matching C
types or qualify recovery after a rejected command.

Offline verification matched every instruction word and both library hashes,
then independently decoded the option-packing instructions and checked their
shift/OR arithmetic. It did not execute the complete functions, emulate the
VPU or test slice-ready behavior. The separate
[complete command-dispatch review](../soc/ar9311-decoder-commands.md) now covers
the caller and its jump table, including SDK command-number differences and
ignored backend returns. These scoped reviews do not close the whole library
audit.

### Observed output

| Test | Observation |
| --- | --- |
| Ordinary full-frame decoding | Hardware H.264/H.265 output matched the tested reference pictures |
| Progressive picture input | Complete pictures decoded correctly, without demonstrated early usable output |
| Forced finalization after 1/2/3 of 4 slices | Six controls across both codecs produced exact supplied color bands |
| Missing regions after forced completion | Matched previous-frame content in the tested controls |
| Additional progressive-input runs | 18 exact completed frames across six runs; no early output-memory changes observed |
| Diagnostic codec firmware patch | A two-byte RAM patch exposed common wait-routine callers while preserving tested decoding |

## Interpretation

The hardware can reconstruct supplied slices when firmware is told to finish an
incomplete picture. That result does not establish resumable decoding: forced
completion ends the picture rather than proving it can remain open, publish
finished regions and accept later slices into the same output.

No trustworthy decoded-row/slice-ready signal was identified. Safe early scan-out
and a reduction in glass-to-glass latency were not demonstrated. An open host
driver and a successful diagnostic firmware patch also do not establish a complete
codec CPU ISA or a source-buildable firmware replacement.

The [codec firmware study](../firmware/wave521c-reconstruction.md) records
the wait-loop addresses, signed-immediate correction, parser observations and
validated diagnostic patch separately from the output experiments.

## Direct HDMI comparison

A later 1920×1080 HEVC demonstration used a gray-blue reference followed by four
colored bands, with slice starts at rows 0/256/512/768. The original unchanged
codec firmware was revision 232676, SHA-256
`99bc74cdf1e7ded785c2b65440c983f4e128727eb2bbdf309b14ddde668fd8ac`.
The two-byte caller-probe patch was not used and had never enabled slice output.

In the same-picture test, target slices were submitted at
10,001/13,001/16,001/19,001 ms. The only target presentation occurred at
19,023 ms, after the last slice, and all bands appeared together. Final pixels
matched SHA-256
`3330d66ea49df1947cc0d2bc109eef5bd3016efa24498fe223c91776fe2c4c5d`.
These submission/presentation times are not calibrated glass-to-glass latency.

The apparent progressive comparison instead used four independent decoder
sessions, each replaying the reference plus a growing target prefix and forcing
completion. One, two, three and four bands appeared roughly 3.1 seconds apart.
All supplied Y/U/V regions matched the target and missing regions matched the
reference. Each session held the prior scanout buffer until the replacement
flip completed. This demonstrates reconstruction after forced finish, not
resumption of an unfinished picture.

The diagnostic DMA display path presented the
hardware-decoded bands without CPU painting. Read-only post-completion dumps
were diagnostic. A warm module replacement that timed out, and a cold insertion
missing dependencies, were excluded because neither produced a valid decoder
run. The standard driver and exact full-frame reference hashes were restored
after the successful experiments.

## What would establish the next step

A positive resumable result needs correct, non-stale pixels while later slices
are withheld; a reliable completion indication for those regions; correct final
output after the remaining data arrives; and repeatability across patterns and
reference pictures. Display tests would then need synchronization with ongoing
device writes and an independent latency measurement.

The experiments make firmware scheduling and output handling useful investigation
targets. They do not prove that hardware continuation, state retention or memory
writeback imposes no further constraints.
