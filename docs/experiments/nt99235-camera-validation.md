# NT99235 capture validation and corrected interpretations

[Experiments](README.md) · [Camera hardware](../soc/ar9311-nt99235-camera.md) ·
[ISP reconstruction](../soc/ar9311-nt99235-isp.md)

## Question and evidence scope

The reference investigation asked whether the open AR9311 camera path produced
new, correctly processed frames from the NT99235, and which parts still depended
on vendor-established state. It combined static library analysis, stock/open
live comparisons, MMIO traces, marked output buffers and rendered images.
The retained notes include an August 14, 2026 tone-selection correction;
individual capture dates are not consistently given. These are reported results
at the reference snapshot, not new measurements or SC231AI P1 qualification.

The narrow stock trace contains 48,183 writes: ISP 34,790, VIF 12,639 and CSI
754. It covers `0x08860000–0x08c0ffff`, with 1,267/74/52 distinct registers
respectively. It captures stores by the instrumented userspace process, not
loads or writes by other processes. Its approximately 2,060 repeated cycles
show buffer-address updates and interrupt clears; they cannot establish a
polling policy or DMA completion by themselves.

A wider trace, 330,630 writes over `0x08000000–0x0a1fffff` with an excluded
subwindow, exposed CVISP and clock programming missed by the narrow capture.
The exclusion followed a destructive earlier sweep, so this was not an
unrestricted safe-readable register map. Separate read/write captures supported
later clock observations. The absence of reads in the original store-only trace
must not be used as evidence that the vendor never polls.

## Establishing a fresh frame

DRAM can retain the vendor's last image across a RAM boot. Reading that image
back from the expected output address can look like a successful open capture.
The reference harness placed 24 `0xa5a5a5a5` marker words, spaced by `0x1000`,
in each plane before capture and counted **overwritten** markers.

| Observation | What it establishes |
| --- | --- |
| 24/24 markers overwritten on each plane | DMA wrote the tested locations during that capture window |
| 0/24 overwritten | No writes at those probes; an attractive dump may be a retained frame |
| Five distinct ring images after one second, 10–16% sampled pixel differences | Sustained changing output within the working session |
| Frame-start/arm counts increase | Input events and software arming; not completed pixel writes |
| Grabber process stays alive | It may simply be blocked waiting for a buffer |

One introductory sentence in the source calls the count “surviving” markers,
but its explicit good/bad measurements use **overwritten** counts. The table
above follows those measured examples. Sparse probes establish writes at sampled
locations, not correctness of every pixel or full-plane coverage.

Later position-dependent whole-buffer markers distinguish active pixels,
stride padding and allocation tails. At runtime, the reported extents are
518,400 luma words and 138,240 words in each chroma plane. The first buffer
marked before stream-on is a poor extent oracle: an already rotating fixed ring
can overlap it and write extra tail rows. Measurements taken while capture is
already streaming avoid that contamination.

## First bring-up, restart failure and later workaround

The historical controlled run produced 24/24 overwritten markers on the first
bring-up. The second and third each reported 241 frame starts and 241 slot arms
in four seconds, with CVISP control still `0x00800806`, but **0/24** overwritten
markers. Successful register setup and plausible frame rate did not establish
output DMA. Those runs began from a RAM boot after the vendor pipeline had been
streaming, so inherited hardware state was a material confounder.

Within a successful bring-up, the five distinct images and approximately
60 fps cadence refute the separate idea that the path only emits one frame.
Regular ring arming is necessary. Neither finding explains every subsequent
full teardown failure.

The inspected CVISP implementation later avoids upstream teardown: the first
capture starts VIF/sensor and ISP, while later capture starts reuse them.
Stopping capture returns DMA to the reserved ring and leaves the sensor running.
Thus **queue reopen and full camera restart are different tests**. The driver
workaround does not prove a recovered CVISP reset or independent cold-start
sequence for all boot states.

Buffer lifetime also limits the latency claim. Depth-one output could pass a
write-completion marker check while producing torn encoded frames; the consumer
was still reading when reuse occurred. Default depth three is the recorded
validated policy. Clean drop counters alone do not establish clean pictures,
and none of these tests measures glass-to-glass latency.

## Sensor agreement and image corrections

The 184-register streaming comparison reached zero differences after adding the
sensor's post-mode shading callback. A broader comparison of 3,328 readable
registers retained differences in status, gain/shadow state, sensor-MCU output
and temperature-related bytes. The temperature bytes rose within either stack;
they were not a configuration mismatch or a calibrated temperature measurement.
An anomalous frame-length byte `0x0340=0x08` came from an accidental I²C write;
the clean value was `0x04`.

Several image defects had concrete ownership or setup causes:

- **Cold-boot coloured blobs:** setup replay replaced owned ISP shading
  descriptors with stale vendor addresses. Republishing after replay removed
  the defect in two independent cold boots. Dark-frame channel differences
  were below about 1.3 counts; radial shading still differed modestly from stock.
- **Crushed shadows:** omitted DE3D runtime words `0x2e2c/0x2e30` lay beyond
  the replay prefix. Restoring them reduced the reported under-luma-32 fraction
  from 57% to zero in the compared test. That is a scene-specific measurement,
  not a general exposure target.
- **Colour mismatch:** an identity CCM1 survived early setup, while the stock
  runtime installed a packed tuning matrix. Recovering that later write fixed
  the configuration discrepancy; static AWB-off tuning still limits the claim.
- **Synthetic-bar edge notch:** later averaging over 3,400 real-scene optical
  edges found no corresponding notch. The retained conclusion attributes the
  synthetic artifact to the sensor's test-pattern generator; it was not evidence
  of an ISP defect in live imagery.

The explicit cold-boot shading tests support that repair. They do not erase the
separate full restart failure or establish every inherited clock/reset dependency.
There was no delivered raw bypass image, so the study could not compare an
unprocessed frame with ISP output from the same open capture.

## Conclusions that remain bounded

The strongest established findings are the corrected sensor configuration,
working CSI timing, sustained fresh processed output within successful sessions,
decoded statistics and extensive operating-point register/table reconstruction.
The following still prevent a claim of complete camera parity:

- Dedicated CVISP completion IRQ, reset ownership and full teardown/restart.
- Raw VIF bypass delivery and general camera clock/reset initialization.
- Continuous gamma/DRC/CM/CM2 selection, anti-flicker and LTM/CLAHE production.
- Broader adaptive-stage hardware qualification, image quality and alternate modes.
- The separately recorded dark-frame left-edge ramp and hard ring contours;
  resolving the synthetic-bar notch did not resolve these other artifacts.
- Conflicting historical descriptor-length and gate interpretations identified in
  the [ISP study](../soc/ar9311-nt99235-isp.md).

The [SC231AI P1 validation](p1-source-stack.md) has its own later source/runtime
results. Its camera restart work must be evaluated against that board's actual
implementation rather than used to silently close this reference investigation.
