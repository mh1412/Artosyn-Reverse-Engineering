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

The [DMA display path](../systems/ar9311-display-pipeline.md) presented the
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
