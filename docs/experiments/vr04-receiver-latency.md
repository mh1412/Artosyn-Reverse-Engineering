# VR04 reference receiver latency and display pacing

[Experiments](README.md) · [Reference board](../devices/betafpv-vr04-reference.md)

## Measurement scope

The retained MissingLynk report describes a July 20, 2026 receiver bench and
later display-pacing analysis, consolidated at userspace revision
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`. Results below are reported measurements;
the original timestamp captures were not independently reprocessed for this wiki.

This measures **first received video datagram to display latch on the goggle**.
It excludes camera exposure, ISP, encoding and transmission before that first
datagram. The latch also precedes the relevant pixel's panel scanout and optical
response. It is not camera-to-screen latency or a measurement of RF delay alone.

All stage marks use the goggle's monotonic clock, matched by PTS/FrameId. No
air/goggle clock synchronization is needed for these local intervals.

The report interprets the DRM event as the latch boundary. The later
[display implementation](../systems/vr04-open-media.md) also retains buffers for
an extra flip because software vblank events can precede the hardware latch.
Thus these are software-instrumented intervals; their exact relationship to
physical scanout needs independent confirmation, even before optical response
is considered.

| Mark | Recorded boundary |
| --- | --- |
| `rx` | First UDP datagram of the FrameId on port 10001 |
| `dec0`, `dec1` | Decoded tiles delivered to appsink |
| `pair` | Both tiles available and the frame assembled |
| `issue` | Entry to the display flip ioctl |
| `submit` | Return from that ioctl |
| `flip` | DRM event for scanout latch at the refresh edge |

Since `rx` is the first packet, receiving the remaining packets is inside the
interval. Tile decode time cannot be isolated by subtracting these marks alone.
Instrumentation emits periodic summaries, with optional per-flip raw timestamps.

## Reported July comparison

Run 0104 is described as a five-minute run with 18,249 frame records. It combines
immediate flip submission with suppression of the HUD's redundant plane commits.

| Receiver interval | Median |
| --- | ---: |
| First packet → tile 0 decoded | 4.5 ms |
| Tile 0 → tile 1 decoded | 8.0 ms |
| Paired frame → submitted flip | 1.1 ms |
| Flip ioctl itself | 0.11 ms |
| Submitted flip → latch | 10.2 ms |
| First packet → latch | **25.1 ms**, p99 **35.3 ms** |

These are distributions of per-frame intervals. Their medians need not add to
the median total. The report's attribution of much of tile separation to air-side
packet pacing is plausible, but these marks combine arrival and decoding; they
do not independently isolate each cause.

| Configuration | First-packet-to-latch p50 | p99 | Reported completed fps | Repeats/min |
| --- | ---: | ---: | ---: | ---: |
| Baseline, run 0098 | 47.5 ms | 67.4 ms | 39.7 | 1280 |
| Immediate submit and HUD suppression, run 0104 | 25.1 ms | 35.3 ms | 59.5 | 135 |

The total frame-record count and rounded five-minute duration do not reproduce
the reported completed-fps denominator. The record does not resolve that
accounting detail, so the table preserves its reported metrics rather than
substituting a rate calculated from potentially different populations.

The diagnosis identified legacy HUD `SETPLANE` calls becoming blocking commits
on the shared CRTC. DRM `stall_checks()` then delayed a video flip until the
overlay commit's vblank cleanup, costing an extra refresh on roughly 54% of flips.
When fresh live-video state is present, video commits already latch the overlay
pixels; the HUD suppresses redundant commits and resumes self-refresh after video
stops. This is suppression of display commits, not removal of the HUD pixels.
The comparison changes two behaviors together and does not isolate each one's
individual latency benefit.

## Conflicting pixel-clock mappings

An early analysis used stock timing totals of 2091×1144 and a nominal 148.5 MHz
pixel clock to obtain 62.079 Hz. Against a roughly 60.33 Hz pairing rate, that
predicts about 105 repeats/min from the beat, compared with 135/min reported.
It is a model of that run, not a fully resolved panel-clock calibration.

A later August 25 grid fit of 1200 flip timestamps conflicts with that mapping:

| Mapping | Effective pixels/frame | Predicted refresh at 148.5 MHz | Refresh span for 141.1–155 MHz |
| --- | ---: | ---: | ---: |
| Fixed timing totals | 2,392,104 | 62.08 Hz | 59.0–64.8 Hz |
| Later flip-event fit | 2,475,371 | 59.99 Hz | 57.0–62.6 Hz |

During the fit the commanded clock was 148.44 MHz; the fitted grid residual was
0.018 of a period. The rows disagree by about 3.5%. A simultaneous requested-clock
readback and VO timing dump was still needed to distinguish changed totals from
the clock leaf landing below its requested rate. The early 62.08 Hz value must
not be presented as an independently settled physical refresh rate.

For a 60.33 Hz source, the two models imply seeds near 144.3 and 149.3 MHz.
The report records 149,317,000 Hz in service on its bench unit; that is a bench
setting, not a recommended value for another panel or the Gnd485 HDMI bridge.

## Pacing control and remaining jitter

The open driver exposes `pclk_hz`, clamped to 141.1–155 MHz with approximately
0.049% steps. The controller uses the fifth percentile of submit-to-latch wait
as its phase observation. A deadband avoids chasing the noise floor, and an
asymmetric clamp limits integrator bias after phase wraps. These corrections
addressed a sawtooth with a repeated frame roughly every two seconds.

Phase observation alone gives poor frequency acquisition. When phase traverses
a whole refresh cycle within a sampling interval, the fifth percentile stays
near 5% of a period regardless of the rate mismatch. The recorded integral walk
was roughly 2.4 kHz/s, so an initial gap around 3% could take tens of minutes to
close. A calibrated seed matters; merely enabling the servo did not establish
prompt lock from the boot rate.

Across about 27,000 paced flips, reported source cadence ranged from 59.68 to
60.59 Hz, with half-second-window changes up to roughly 1.2 Hz. Arrival jitter
around the local trend was p90 7.2 ms and p99 19.4 ms. The report estimates a
remaining floor near 30 repeats/min from late arrivals. That is an inference
for this workload, not an unavoidable rate for all links.

Pacing reportedly bounded receiver latency near 19–30 ms rather than a slow
24–40 ms drift. These ranges are not another version of the July percentile
comparison and should not be merged into one benchmark. An additional buffered
refresh was proposed to absorb jitter at about 16 ms latency cost; no measured
smoothness result for that mode was supplied.

The remaining work is to reconcile clock/timing readback, preserve raw metric
denominators, measure upstream cadence under controlled source settings and add
an optical camera-to-screen test. The
[synthetic air benchmark](vr04-air-encoder-benchmark.md) measures a different
boundary and cannot fill those missing latency stages.
