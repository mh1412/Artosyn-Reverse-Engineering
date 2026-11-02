# NT99235 ISP startup, memory ownership and runtime controls

[SoC studies](README.md) · [ISP reconstruction](ar9311-nt99235-isp.md) ·
[Camera path](ar9311-nt99235-camera.md)

This page describes the open ISP implementation at the
[pinned MissingLynk revision](../references/README.md#platform-and-application-references).
It follows the main driver, table allocator/appliers and register definitions.
These are source-level contracts and limitations, separate from the
[recorded camera experiments](../experiments/nt99235-camera-validation.md).

## Startup success and degraded operation

The exported pipeline-start function applies the configured setup prefix
(1,475 entries by default), then a separate output-arm sequence. The full-table
debugfs configuration path is a different operation: it also applies the trim
pass, and its historical input-loss problem is why it is not the default start.
Reading `configure_upto` reports the table's total size, not the last prefix
applied. Neither path reproduces the vendor's original interleaving of ISP,
CSI and VIF writes.

Several failures do not propagate through pipeline start:

| Condition | Implemented behavior |
| --- | --- |
| No ISP instance | Returns `-ENODEV` |
| No 32-bit DMA mask or reserved-memory attachment | Table preparation returns early; probe continues, and tuning is not loaded |
| One coherent allocation fails | Warns, leaves that buffer absent and continues allocating the others |
| Tuning missing or wrong length | Leaves tuning unavailable; configuration continues |
| IRQ unavailable or request fails | No working statistics handler is established; pipeline start still returns success |

Where an owned buffer is absent, its publisher skips the address replacement,
so replayed descriptors can remain. This behavior is inherited-state fallback,
not evidence that a camera works from a cold boot in those conditions. The
`configured` flag means the configuration routine returned; it is not a
pixel-output, allocation-completeness or IRQ-health check.

The tuning loader requires exactly `0xd6c58` bytes and permits a device-tree
firmware-name override. It does not verify the file's hash or sensor identity.
Equal length therefore does not make another sensor's tuning interchangeable.
Several ownership switches govern allocation at probe; changing them afterward
does not allocate a missing buffer. Likewise, `tables=false` skips the common
fill/publish pass but is not a global hardware stop: output arm has separate
statistics and DE3D publication calls.

## Coefficient ownership and inherited data

With `seed=true`, the gamma, DRC and shading fill paths first copy residual
vendor memory, even when a later tuning-derived fill succeeds. The generated
gamma payload and complete DRC page then replace their relevant regions.
Allocation tails and unrecovered shading content have different provenance.
With seeding disabled, the corresponding allocations are cleared before fill.

The compander uses its carried template, while its shared HDR prefix is cleared.
HDR shading has no recovered builder: it is seeded or cleared. The LTM page
starts as an identity curve. Buffer ownership therefore does not by itself
establish that every byte was reconstructed from tuning, or that every stage's
adaptive algorithm exists.

Configuration fills memory before a write barrier and descriptor publication.
This orders CPU writes; it does not establish that a previous hardware fetch
has finished. Removal first releases the IRQ and removes diagnostic files,
then frees coherent buffers and disables clocks. No explicit ISP DMA drain or
engine-stop sequence appears in that removal path. Interrupt quiescence and
DMA quiescence remain separate questions.

## Statistics event and export format

The IRQ handler reads and writes back the two W1C status words at `0x00cc` and
`0x00d4`. Status word 1 bit 8 drives the CVISP frame callback and, when enabled,
statistics-buffer alternation. The handler calls CVISP before flipping the
statistics halves. IRQ registration is deferred until output arm to avoid the
recorded interrupt storm while input arrives at an unconfigured ISP.

RRO, smaller-window RRO and raw histogram each have two halves. Publication
resets the current/readable indices and clears validity; the next statistics
event arms the other half, publishes the completed index and increments
`stats_flips`. It does not reset that sequence counter. With ping-pong disabled,
readers receive the live first half and the sequence no longer tracks writes.

The `stats_raw` diagnostic record is **`0x5808` bytes** on this little-endian
target:

| Offset | Size | Content |
| --- | --- | --- |
| `0x0000` | 4 | Starting `stats_flips`, native `u32` |
| `0x0004` | `0x4800` | Completed RRO zone-grid half |
| `0x4804` | `0x1000` | Histogram allocation, including its undecoded/unused tail |
| `0x5804` | 4 | Ending `stats_flips`, native `u32` |

Unavailable buffers or no completed ping-pong frame produce `-EAGAIN`.
The record contains no sensor timestamp, smaller-window RRO or LTM histogram.
The intended reader rejects unequal sequence words and retries. Every retry
needs the same check; a second attempt is not guaranteed coherent. Export does
not hold the statistics lock across the copies, and equality is a software
sequence check rather than an atomic DMA snapshot or protection against a
concurrent reconfiguration.

LTM statistics remain single-buffered: `0x8000` bytes form the decoded histogram
head of a `0x80000` allocation. Reads can span hardware updates. The tail-map
diagnostic counts nonzero bytes in 4-KiB blocks; a zero block is not proof that
hardware never wrote it, nor is a nonzero block a decoded data structure.

## Tone and LTM controls

The `ladders` control reapplies RNR, LNR, DE3D, CFA, CNF, CM and CM2 using their
current parameters, then reapplies DE3D geometry. The first five use gain;
CM/CM2 use their separate tone-trigger parameters. Changing a parameter alone
does not call these appliers.

The separate `tone` control rebuilds gamma/DRC: value 1 skips an unchanged
selection, while values greater than 1 force rebuilding. `tone_scalar=-1`
selects the pinned curve/profile indices; a nonnegative scalar selects and
blends using the tuning bands. This hook neither updates the CM/CM2 parameters
nor proves an automatic scalar producer. It writes the existing gamma/DRC
allocations in place, then publishes descriptors after a barrier; there is no
inactive-page swap, frame-boundary wait or mutex in this path.

LTM uploads use a different mechanism. Each open file has a zeroed `0x4000`
staging buffer. Closing publishes if the greatest successfully written end
position equals the page size. **This is a high-water-mark check, not proof that
exactly one contiguous page was supplied:** seeking to the last byte and writing
it can qualify with zero-filled holes. An oversized write is capped at the
remaining buffer space; reaching the end still qualifies for publication.
A nonempty short feed is dropped with a warning. The close handler ignores the
publisher's return value, so successful close alone does not prove installation.

The publisher locks, copies into the opposite coefficient half and redirects
`0x2808`. It has no frame-completion wait. Ordinary feed publishers serialize,
but configuration fills both halves without taking that mutex. Reads lock each
call rather than retaining one page snapshot across multiple calls. Concurrent
reconfiguration or repeated uploads therefore need their own validation;
double buffering alone is not a general concurrency guarantee.

## What the diagnostic comparisons establish

The `ladder_banks` report compares live registers with the same builders used
by the appliers. It is useful for readback agreement, not an independent proof
of the builders' arithmetic. LNR starts its expected values from the live bank,
so untouched fields agree by construction. The DE3D comparison omits the two
optional strength transforms applied by the writer; non-neutral strength can
therefore produce a mismatch even when the requested transform was installed.
The CNF comparison covers its three ladder/normalization words, not every static
word its applier writes. Skipped stages are not successful parity checks.

The gate report has similar limits: unresolved polarity, missing tuning flags,
hardware-owned registers and whole-word gates do not receive the same verdict
as recovered bit gates. The optional veto is off by default and counts deviations;
enabling it can write changes. A zero mismatch count does not qualify every
stage or supply missing DMA allocations.

The offline top-level-bank checker passed against the pinned library. It checks
geometry masks, selected instruction constants and template words, plus a
hard-coded enable-bit composition. Its input is the full setup/trim table model,
not a capture of the default prefix/output-arm execution. Matching constants
within disassembly windows is also narrower than tracing their complete control
flow and register destinations. This check does not establish live frame output
or close the remaining provenance and restart work.

The older defaults generator's copy-agreement helper also has a narrower scope
than its prose suggests. It extends outward from the traced span while copies
agree, but does not compare the copies inside that initial span. A synthetic
two-copy fixture with one differing interior word still returned the whole
64-register page as verified. This demonstrates a checker's blind spot, not an
observed mismatch in the pinned real library. The resulting static-defaults
array is retained for reference and is not the table applied by the driver;
actual template lengths and recorded write order still need separate evidence.
