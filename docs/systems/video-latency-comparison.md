# Ascent latency: release boundaries, queues and display timing

[Systems index](README.md) · [Video paths](video-paths.md) ·
[Ascent ground station](ascent-groundstation.md)

The October 12, 2026 investigation identifies latency-related mechanisms and a
specific Ascent exporter release delay. It does **not** contain a matched optical
glass-to-glass benchmark of local HDMI and network viewing. Its strongest direct
result is a source-level parser experiment, repeated during this review. That
result describes network export, not the stock HDMI decoder.

A later, separate [VRX Pro beta4 report](../experiments/vrx-pro-latency.md)
contains optical measurements for a Lite+/RK3568/120-Hz-panel bench. It does
not supply the missing matched local-HDMI-versus-network comparison here.

## Compare the same endpoints

| Path | Scope of the evidence |
| --- | --- |
| Ascent local output | Standard Gnd485 / AR9311, SDIO radio, Linux daemon, local TCP SDK, stock backend, hardware media output and external HDMI display |
| Ascent network viewer | The same receiver ingress, plus copied-data worker, vendor/NAL/picture assembly, RTP over RNDIS, host decoder and desktop/display scheduling |

The Ascent comparison used the historical 17.5.3 application with selected
17.5.8 continuation components. The Sky482/Lite 17.5.8 transmitter was an offline
reference, not identification of the active camera or encoder settings. The
captured stream was 1920×1080 HEVC Main at nominal 60 fps. Those facts do not
automatically describe later Open8030, fpvOS or other boards.

The Ascent RTP worker is a side branch: stock TCP forwarding does not wait for
its picture assembler.

Glass-to-glass delay runs from a scene change to corresponding light at the
display. It includes exposure/readout, processing, encoder release, transport/
recovery, receive queues, decoding, presentation, scan-out to the measured pixel
and panel response. Stages can overlap; adding their entire frame-processing
times can overestimate the critical path. Throughput, picture age and recovery
time are separate: an old-picture FIFO can still output 60 frames per second.

## The verified Ascent exporter boundary

The reviewed source has three nested release conditions:

| Stage | Release condition | Consequence |
| --- | --- | --- |
| Vendor wrapper | Complete payload and `0xbb` terminator | Earlier video bytes wait for the enclosing message |
| Annex-B NAL parser | A following start code | The preceding NAL is not classified immediately after its last byte |
| Picture/access-unit assembly | Classification of a new first-slice VCL NAL | Normal previous-picture closure depends on the next picture |

An access unit is the assembled coded picture. Parameter changes can separately
finish the preceding picture using its old configuration. For ordinary VCL
progress, the next first-slice flag is available after a few header bytes, but
classification waits until a following start code finishes that NAL.

The probe supplies synthetic parameter headers, an IDR-like picture and a second
first-slice NAL in separate chunks:

| Input checkpoint | Completed-access-unit decisions |
| --- | ---: |
| Picture 0 bytes supplied, no following start code | 0 |
| Picture 1 start code supplied | 0 |
| Picture 1 first-slice header/body supplied | 0 |
| Following start code supplied, permitting picture 1 classification | 1 |

It uses small synthetic headers, **no UDP destination and no HEVC decoding**.
The counter increments before network-output gating; it is not a count of
transmitted or decoded pictures. Fresh compilation reproduced `0, 0, 0, 1`.

With one VCL NAL per picture and no intervening delimiter, picture `n` can remain
pending until the start code at `n+2`: the start of `n+1` finishes parsing `n`,
and the following start finishes/classifies `n+1`, closing `n`. At 60 fps that
can add approximately a frame interval after `n`'s last bytes, with residence
from first bytes spanning roughly two picture starts. This is an illustrative
arrival pattern, not measured delay for every live picture. Slices, metadata/SEI,
wrapper release and burst timing change the interval.

The exporter then packetizes all stored NALs with one RTP timestamp and a final
packet marker. It does not send each slice as it arrives. If the next delimiter
never comes, partial data can remain pending while TCP stays open. Shutdown
discards unfinished tails; there is no age-based flush deadline. A timer cannot
safely declare arbitrary partial bytes to be a complete picture.

Checked source identities:

| Component | SHA256 |
| --- | --- |
| Video parser/assembler | `92ba20302e7586b02000f98409ebd6bcf8b2720ebe847e4b40f560f905c7f906` |
| Vendor-message parser | `8249567e1701454bdf7c0f4526b59218d02b253dfaa5b8a68a3fb777932fa01e` |
| RTP packetizer | `1febc3f6ae4f1f523703b3d9906eb7eacec399b3de62fae449ca718e33903459` |

These identify inspected source, not the installed ARM64 executable. This review
did not change the exporter or measure device latency.

## Bounded memory does not bound picture age

The preallocated single-producer/single-consumer tap FIFO has 127 usable slots
in its default 128-slot ring, each holding up to 8 KiB. It drops newly copied
input when full; it does not evict the oldest picture on an age deadline. Byte
offset gaps reset parsing and recovery-picture gating while preserving stock
forwarding.

At the historical scene's approximately 12.7 Mbit/s payload rate, these are
**calculated amounts of data**, not observed occupied queues or fixed waits:

| Capacity | Equivalent data duration at 12.7 Mbit/s |
| --- | ---: |
| One 64 KiB TCP relay buffer | 41.283 ms |
| 127 × 8 KiB tap slots | 655.360 ms |
| Requested 256 KiB viewer receive buffer | 165.130 ms |
| One maximum 8 KiB tap chunk | 5.160 ms |

The calculation is `bytes × 8 / bitrate`. Chunks vary, nonvideo traffic can enter
the ring, and socket accounting differs from compressed-payload occupancy.
The 1 MiB wrapper/NAL and 4 MiB picture limits constrain size, not freshness of
smaller incomplete objects.

Three surface-sized Ascent framebuffer regions do not imply three frames of
delay. Surfaces may serve DMA, prediction,
scan-out or a newest-pending picture rather than a display FIFO. Track picture
IDs, oldest age and release/retirement. Arbitrarily dropping compressed bytes or
reference pictures can corrupt later pictures; stale decoded presentation work
has a different recovery and ownership boundary.

The proxy already requests `TCP_NODELAY`. Its `poll(..., 100)` is a maximum idle
wait, not 100 ms per packet: readiness/eventfd wake the loops. Its 8 KiB receive
limit is a maximum, not a fill requirement. UDP sends are nonblocking. These
facts rule out superficial diagnoses without proving negligible scheduling cost.

## Codec, bus and clock interpretations

Ascent's HEVC Main capture or `TRAIL_R` NAL type does not establish B-picture
reordering. Reference structure, lookahead, encoder release, rate control and
decoder/output queues need their own evidence. One viewer decode thread does
not prove hardware acceleration or throughput headroom. Later
[AR9311 slice experiments](../experiments/ar9311-slice-decoding.md) produced
selected regions by ending an incomplete picture, but did not establish
resumable slice decoding or safe partial-frame scan-out.

At 12.7 Mbit/s and 60 fps, average compressed size is about 26.46 kB. Assuming
four-bit 50 MHz SDIO and USB 2 high speed, ideal serialization is about 1.06 ms
versus 0.44 ms before overhead. Effective SDIO rate and negotiated USB speed were
not measured in this comparison; the arithmetic cannot establish a real bus-delay
advantage. DMA, batching, queueing and wakeups matter. Moving raw 1080p60 pictures
would instead mean 186.624 MB/s for YUV420 or 497.664 MB/s for ARGB8888, before
extra copies; the exporter forwards compressed data without re-encoding.

RTP timestamps derive from frame-ID progress and integer FPS. Vendor timestamps
help detect discontinuities but do not establish camera-to-host clock mapping.
`ffplay -sync ext` selects a local playback clock, not hardware synchronization
to the air unit. Low-delay flags and frame dropping cannot recover time already
spent in the exporter; startup probing is separate from ongoing frame age.

One 60 Hz period is 16.667 ms; a uniform independent phase wait averages 8.333 ms.
At 100 Hz these are 10 ms and 5 ms. Higher camera/display rate can reduce waits
while increasing processing/RF pressure. A sustained 60.0024 fps source would
differ from exactly 60 by about 40 ppm, but the recorded estimate was not a
calibrated long-term oscillator measurement.

## Recovery and other branches

The exporter caches parameters and waits for IDR/BLA types 16–20. It has no CRA
bootstrap, replayable GOP or automatic keyframe request. The historical mapped
stock I-frame request was untested in this comparison; later
[native keyframe ACK experiments](../experiments/ar8030-first-video.md) have a
separate version and scope. Joining, steady-state age and RF-loss recovery are
different problems. The recorded no-picture incident with no compressed ingress
was upstream of this assembler.

Large refresh pictures can cause bursts at moderate average bitrate. For
illustration, a ten-times-average picture at 12.7 Mbit/s and 60 fps takes roughly
105.8 ms through a 20 Mbit/s effective link, versus 10.6 ms for an average one.
Neither burst factor nor link capacity was measured here. GDR and on-demand
refresh require coherent decoder/reference recovery.

Qt branding, thread count, ION/MMZ allocation and graph fan-out do not establish
CPU-painted video, scheduler delay, zero-copy transport or complete branch
isolation. Recording can retain shared surfaces; UI and export can contend for
resources outside the main decode graph. The older decode–H.264 encode–decode
preview was removed after reported extra delay; it is not the direct HEVC path.

## Experiments still needed

Compare stock local HDMI, proxy-enabled local HDMI and direct HEVC host playback
separately. Match exposure, resolution, picture rate,
lighting, RF conditions, recording state and image row. Optical scene-trigger
tests should report distributions and frozen/lost-picture intervals, including
top/center/bottom rows. Decode completion omits presentation and panel delay.

Internal traces should connect picture identity to ingress first/last bytes,
enqueue/dequeue age, wrapper completion, first-slice recognition, NAL/picture
closure, RTP packets, decode submission/completion and display acceptance/flip.
Use one monotonic clock within a host; validate cross-clock mappings before
subtracting timestamps. Coarse UI telemetry is not video timing.

The concrete exporter investigation is incremental first-slice recognition, then
earlier packetization where boundaries permit it. Preserve parameter changes,
interleaved/missing metadata, RTP markers and loss recovery. An age budget needs
codec-aware resynchronization rather than arbitrary byte drops. For stock HDMI,
trace encoder release and VDEC-to-display ownership; the exporter result cannot
explain that branch. Worker stalls, recording/UI load and RF loss need separate
tests of picture age and recovery, not just average FPS.

The parser probe and selected frame-period, capacity and raw-pixel-rate
calculations were checked again. Those checks added no radio operation or
optical measurement.
