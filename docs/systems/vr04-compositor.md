# VR04 compositor memory and seam handling

[Systems](README.md) · [Frame ownership](vr04-frame-ownership.md) ·
[Recording](vr04-recording-restream.md)

## Scope

This review covers the complete composite allocator/copy implementation, seam
arithmetic, shared pipeline declarations and seam test at MissingLynk revision
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`. The dynamic runtime launcher and its
environment setup were also checked. These source findings do not extend the
[retained DMA hardware tests](../experiments/vr04-media-validation.md#related-dma-and-usb-failures)
to every fallback or failure path.

## Heap selection and actual requested sizes

The allocator prefers `ML_HEAP` when set, otherwise `mmz`. If that heap cannot
be opened, it scans the heap directory and uses the first entry it can open.
It does not try another heap after an allocation ioctl fails on the chosen
one. Thus an override is a preference rather than a strict heap requirement,
and directory fallback is not a specified MMZ-to-CMA priority order.

The checked launcher forces composite mode but does not set the heap or pool
cap. Its comment claiming the binary defaults to CMA is stale: the current
binary prefers MMZ. The environment helper does not override that choice.
Earlier measured full-depth CMA runs remain evidence for those configurations,
not proof of the heap selected by every later launch.

The packed composite is I420 with 1920-byte luma and 960-byte chroma strides.

| Quantity | Bytes | Meaning |
| --- | ---: | --- |
| Luma plane | 2,073,600 | 1920 × 1080 |
| Each chroma plane | 518,400 | 960 × 540 |
| U / V offsets | 2,073,600 / 2,592,000 | Packed plane placement |
| Composite image | 3,110,400 | Bytes containing the 1080p picture |
| Composite allocation request | 3,133,440 | Encoder size-image capacity for 1088 rows; plane offsets remain packed |
| Staging allocation request | 3,110,400 | One maximum-size packed I420 image |
| One seam band | 92,160 | 32 luma rows plus two 16-row chroma bands |
| Four-region seam scratch ring | 368,640 | 360 KiB, allocated for blend mode |
| Four 720p scale destinations | 5,529,600 | Maximum recording-scale pool request |

The staging and scratch values supersede nearby comments saying “1.6 MB” and
“180 KiB.” These are requested payload capacities; heap alignment, page-table
cost, imported framebuffer ownership and other media allocations are additional.

Initialization claims staging, seam scratch and recording-scale buffers before
filling the composite pool. The pool stops at the first allocation or mapping
failure and requires at least five buffers. `ML_COMP_MAX` is clamped to 5–24;
without it the upper limit is 24. A partially allocated pool is not a complete
memory budget or a guarantee that subsequent decoder allocation succeeds.

The input gate derives `window = max(4, pool_count − 6)`, then divides it into
skew and in-flight limits. Counts 5 and 10 both yield 2/2; 14 yields 4/4; 24
yields 9/9. The minimum window can exceed the buffers left after display holds
four. These are flow-control settings with eviction/drop escape paths, not a
proof that pairing always fits. `ML_NO_RECYCLE`, when present, deliberately
prevents returned composite buffers from re-entering the free list.

## Copy paths and cache assumptions

Direct DMA requires a full-width, packed I420 tile in one DMA buffer. It submits
three contiguous plane copies and can skip source overlap rows. Non-DMA or
row-strided I420 can instead be packed by the CPU into the staging buffer and
then DMA-copied into the composite. NV12 reaches the CPU fallback, which
deinterleaves chroma into I420. Destination width/height clipping does not add
format conversion, rescaling or clearing of uncovered areas.

The staging path issues a cache flush before DMA but ignores that flush's
result. It returns success when the subsequent DMA submission succeeds, so
“DMA copy succeeded” does not independently prove the staging writes were
visible. Generic whole-buffer sync helpers likewise ignore ioctl results.
The [kernel/DMA review](../experiments/vr04-media-validation.md) explains why
successful software completion and confirmed device quiescence remain separate.

Scratch allocation prefers `default_cma_region` for CPU reads. Its cached/WC
classification tests whether the returned pathname equals the MMZ heap name;
it does not query an independent cache-policy property. Blend mode also probes
a ranged cache-clean ioctl. Allocation or probe failure downgrades to split
mode, although an allocated scratch buffer can remain retained after a failed
probe.

## Overlap modes and geometry

| Mode | Composite behavior |
| --- | --- |
| Off, default | Both tile copies cover the overlap at the legacy placement; the later copy can overwrite earlier overlap pixels |
| Split | Tile 0 retains the overlap; tile 1 skips its first 32 source rows and begins after tile 0 |
| Blend | Split placement plus a cross-fade between both reconstructions of the overlap |

Split geometry requires full-width tiles, positive even heights above 32, and
`height0 + height1 − 32 = 1080`. The normal 560/552 pair puts the band at
composite rows 528–559. Geometry is rechecked as each tile arrives, using the
most recently observed shape for its peer. This is not a proof that both shapes
belong to one new stream configuration or that decoder reconfiguration works.

Tile 1's overlap is captured into one of four scratch regions, selected by
pairing-slot index modulo four. A PTS stamp rejects a region overwritten by a
different pair. More than four outstanding pairs can consequently lose their
blend opportunity without losing the whole composite. Staged/CPU overwrite
fallback and incomplete initial geometry can also make a pair ineligible.

## Cross-fade arithmetic

For each plane, let `N` be the band's row count, `A` tile 0's sample, `B` tile
1's sample, and `w` the row's tile-1 weight:

```text
output = floor((A × (N − w) + B × w + N/2) / N)
luma:   N = 32, w = 0, 1, …, 16, 18, …, 32
chroma: N = 16, w = 0, 1, …,  8, 10, …, 16
```

The ramp deliberately skips weight 17 or 9 so both ends are entirely one
tile. At these band sizes, the rounded numerator fits in 16 bits; NEON performs
the weighted sum and shift without needing a saturating output. The destination
may alias tile 0's band. Unsupported row counts return without blending.

This smooths a boundary between two reconstructed images. It does not repair
missing reference pictures, establish equal encoder state or guarantee the
absence of visible artifacts elsewhere in the frame.

## Failure after blending does not discard the picture

The blend path first invalidates the composite's three band ranges and, for
cached scratch, its captured band. A failure here skips CPU blending. It then
modifies the composite in place and cleans the three destination ranges,
retrying the clean sequence once.

If both clean attempts fail, the function logs once, increments its cache-skip
counter and returns false. **The CPU writes remain in place.** The caller
increments the general seam-skip counter and still submits the completed
composite to display/recording. It does not restore the original pixels or
discard that frame. A “skipped blend” count therefore includes a different
condition from “no blend writes occurred,” and does not certify cache coherence.

A host probe using the exact orchestration functions and mocked arithmetic/cache
operations reproduced this distinction: pre-blend invalidate failure made no
blend calls, while post-blend clean failure retained all three simulated plane
writes and returned false after two attempts. A mismatched scratch PTS also
skipped before writes. This checks control flow, not real cache visibility.

## Divergence measurements and test coverage

Optional statistics measure the absolute and signed difference between the two
bands before blending. Sample counts weight luma by two thirds and each chroma
plane by one sixth. Means are stored at 1/256-sample precision; the signed value
is tile 1 minus tile 0. Sampling occurs only after geometry, stamp and cache
checks succeed, normally once per 60 eligible pairs. It is not necessarily once
per second when frames are lost or blending is skipped.

The statistic compares reconstructed samples. Unlike the source comment's
“free of any scene content” wording, interpretation still depends on the scene,
compression and whether the inputs represent the same content. Zero difference
on static or identical data does not establish fresh, synchronized camera frames.

The retained test checks all 256×256 sample pairs across both weight tables,
compares a scalar vendor-expression transcription with two NEON formulations,
checks in-place luma blending and a black/white ramp, and tests measurement
with different strides and a scalar tail. It does not run the vendor binary,
exercise DMA/cache ownership or establish optical results. Its blend checks use
vector-aligned widths; the explicit nonaligned-tail test belongs to measurement.

The unchanged arithmetic source and test cross-compiled successfully here with
AArch64 warnings-as-errors. They were **not executed**, because the QEMU user
emulator was unavailable. The project's optional test target also reports a
skip when that emulator is absent, so overall test success alone cannot prove
this arithmetic test ran. The separate mocked failure-path probe did execute
and passed AddressSanitizer/UndefinedBehaviorSanitizer checks.
