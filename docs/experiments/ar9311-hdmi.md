# Gnd485 HDMI bring-up and artifact investigation

[Experiments](README.md) · [Display pipeline](../systems/ar9311-display-pipeline.md)

The October 26–27, 2026 investigation established fixed-mode cold-boot output,
live page flips and DMA playback. It also found a reproducible register-bank
access defect. Visible line artifacts and intermittent black capture intervals
remained unresolved after that defect was fixed.

## Controls and first working output

The early prototype's source vblank approached 60 Hz and the bridge measured
1920×1080/1125 input, yet capture showed corrupted horizontal bands. Timing
registration alone did not establish usable HDMI output.

A matched stock 18.21.10 control used the same receiver, cable, MACROSILICON
`345f:2130` capture adapter and 1920×1080 MJPEG30 settings. Stock UI was clean,
and a red/green/blue framebuffer patch changed live before the original bytes
were restored. This showed that the capture path could carry those pixels.

The stock-timed source profile and bridge sequence then produced clean bars on
the first cold modeset. Two-buffer red/green changes verified live output.
One run completed 10,800 flips at about 60.0088 Hz. Normal packaged images
repeated the result without diagnostic MMIO or an internal bridge test pattern,
and a second cold boot produced 1,800 further live events.

Concurrent eight-frame decode checks retained exact reference hashes:

| Fixture | SHA-256 of decoded output |
| --- | --- |
| HEVC 1080p | `e6f33196a402a9fe48546e7f5f957afafd7386eef1422fb90125cdbae4635e82` |
| H.264 720p | `8930563fce44ba5410b0fd58cbc9bda27b0c39e11e9bbc9df29736034f9c1ac5` |

Successful restart evidence required actual temporal changes. A launch that
raced another DRM master and an attempted run after the USB host address vanished
were excluded. A black recording with no successful application launch was not
counted as display failure or recovery success.

## DMA playback and observed artifacts

Cold-boot DMA playback produced correct decoded pixels and live video. A final
HEVC 360-frame run had no anomalies in its sampled static edge region, while
the matching H.264 run had six among 691 checked capture frames after startup.
Some faults appeared while holding the final frame after decoder STREAMOFF.
Later static RGB and independently CPU-initialized buffers also reproduced
artifacts with decoding and page flips stopped. A decoder-only or slice-input
explanation therefore did not fit all observations.

The 60 fps tests presented 600 frames per codec with no skipped steady vblank
gaps. Valid 60 fps capture portions contained consecutive source counters, but
black intervals interrupted independent observation. At 30 fps capture, HEVC
mostly stepped by two source counters; H.264 also had a long black interval.
The latter disproved the suggestion that blackouts occurred only in the adapter's
720p60 capture mode.

Adding a retained-buffer guard did not remove the artifacts. Raising the VO FIFO
threshold from `0x3c0` to `0x400` with stride 1920 caused major corruption and
was rejected. Matching stock unity scaling also did not establish a fix. Sampled
DSI error registers stayed zero during some fault-bearing captures, which cannot
exclude unsampled or unreported downstream problems.

## Measurement boundaries

The corrected detector recognizes the adapter's uniform RGB16 no-signal frames.
Earlier console counts used a threshold that missed them; corrected per-frame
results superseded those counts. Row detection sampled x=16–31, y=90–1079,
with a separate white-bar alignment check. It did not inspect every pixel.

Representative corrected captures were:

| Capture | Initial frames skipped | Nonblack frames checked | Row anomalies | Black frames |
| --- | ---: | ---: | ---: | ---: |
| Initial baseline | 60 | 1,750 | 6 | 141 |
| Independent reference RGB | 60 | 1,744 | 12 | 147 |
| Bridge internal ramp | 60 | 2,641 | 0 | 0 |
| Stock reference | 60 | 2,600 | 0 | 41 |
| Fresh native baseline | 60 | 1,756 | 15 | 135 |

Stock's black frames ended at frame 100, about 3.33 seconds, with no later row
events or blackouts during that 90-second capture. The bridge's internal ramp
was clean but used different content and bypassed the normal input path.
Initial acquisition remained counted whenever it extended beyond the skip window.

Tests of capture-first startup, HPD-triggered recalibration, clock gates, drive
strength, receive equalization, settle timing and blanking packets did not
produce a repeatable causal fix. Quiet intervals failed repeat or reversal checks.
One intended reset reversal never ran because its module was absent; that trial
cannot establish reversal evidence. A temporary autolink trial also used the
wrong HPD condition; corrected OR-condition repeats still glitched.

Pausing the stock backend after acquisition left its image clean; reconnecting
capture while paused reproduced short faults. This motivates acquisition/recovery
investigation but does not prove the native fault's cause.

## Confirmed page-selector defect

The bridge can return stale data when reading selector register `0xff`:

1. Reading AVI header `0x8440` returns `0x82` while selecting bank `0x84`.
2. Reading the selector then returns stale `0x82`, rather than actual bank `0x84`.
3. A later request for bank `0x82` can skip the required write because regmap
   believes that bank is already selected.
4. Reading vertical total `0x826c/0x826d` consequently yields `0x0065` instead
   of `0x0465` (1125).
5. Forcing page writes restores `0x0465`; reversing the change reproduces the
   bad value.

Seven malformed timing samples were traced to this access bug, not demonstrated
changes in the physical input timing. The fix caches only the selector and
leaves data registers volatile. An unused cached initial page forces an explicit
selection after reset. The current implementation makes this an opt-in bridge
property, separate from the fixed video-sequence property.

Cold boot with the fix returned the correct selector and vertical total in all
four checks. Its 90-second static capture still recorded eight row events and
176 black frames after a seven-second skip. Forced-write comparison runs glitched
on both sides too. The register-access repair is therefore established; an
HDMI artifact cure is not.

## Remaining conclusions

Changing capture rate did not isolate the fault: initially quiet 50 and 30 fps
runs were followed by fault-bearing repeats at both rates. The output remained
1080p60 throughout. Capture frame indices must use the selected capture rate;
they are not source-frame or latency measurements.

On October 27, the user reported no visible defects during playback on a monitor
connected directly to the receiver. That supports sink/capture interoperability
as a leading explanation, but it is a reported visual observation, not a captured
signal-quality measurement. Marginal source timing tolerated by the monitor
remains possible.

Later October 29–30 checks still found capture blackouts and inconsistent HPD.
A longer stock control showed first picture after about 3.3 seconds, explaining
why some three-second captures were too short; its final 20-second run had 100
startup-black frames followed by 500 visible frames. fpvOS forced playback still
had mid-playback black intervals. A subsequent shared-display build produced
visible unforced HEVC with a sampled thin-row anomaly while an H.264 capture was
black and its unforced retry reported disconnected. Those are mixed outcomes,
not a clean HDMI acceptance result.

The evidence cannot conclusively distinguish an output-signal defect from
capture-receiver interoperability. An independent timing measurement would help
separate them. Clean pixel dumps, valid vblank events, zero sampled DSI errors
and repaired register reads must each retain their own narrower meaning.
