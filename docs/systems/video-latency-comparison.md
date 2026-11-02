# DJI and Ascent latency: release boundaries, queues and display timing

[Systems index](README.md) · [Video paths](video-paths.md) ·
[Ascent ground station](ascent-groundstation.md)

The October 12, 2026 comparison identifies latency-related mechanisms and a
specific Ascent exporter release delay. It does **not** contain a matched optical
glass-to-glass benchmark or establish a numerical DJI advantage. Its strongest
direct result is a source-level parser experiment, repeated during this review.
That result describes network export, not the stock HDMI decoder.

A later, separate [VRX Pro beta4 report](../experiments/vrx-pro-latency.md)
contains optical measurements for a Lite+/RK3568/120-Hz-panel bench. It does
not supply the missing matched DJI-versus-Gnd485 comparison here.

## Compare the same endpoints

| Path | Scope of the evidence |
| --- | --- |
| DJI integrated goggles | Goggles 3 / ZV902 v01.00.1400 receiver configuration and binaries; O3 compatibility maps to WM1695; air-unit camera/encoder settings uninspected |
| Ascent local output | Standard Gnd485 / AR9311, SDIO radio, Linux daemon, local TCP SDK, stock backend, hardware media output and external HDMI display |
| Ascent network viewer | The same receiver ingress, plus copied-data worker, vendor/NAL/picture assembly, RTP over RNDIS, host decoder and desktop/display scheduling |

The Ascent comparison used the historical 17.5.3 application with selected
17.5.8 continuation components. The Sky482/Lite 17.5.8 transmitter was an offline
reference, not identification of the active camera or encoder settings. The
captured stream was 1920×1080 HEVC Main at nominal 60 fps. Those facts do not
automatically describe later Open8030, fpvOS or other boards.

The configured DJI primary graph connects WL decode through a YUV splitter to
the display/blending sink. GDC and MCTF definitions are absent from its edges;
virtual-view processing is separate. The Ascent RTP worker is also a side branch:
stock TCP forwarding does not wait for its picture assembler. See
[DJI receiver architecture](dji-receiver-comparison.md).

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

Seven DJI source buffers or three surface-sized Ascent framebuffer regions do
not imply seven or three frames of delay. Surfaces may serve DMA, prediction,
scan-out or a newest-pending picture rather than a display FIFO. Track picture
IDs, oldest age and release/retirement. Arbitrarily dropping compressed bytes or
reference pictures can corrupt later pictures; stale decoded presentation work
has a different recovery and ownership boundary.

The proxy already requests `TCP_NODELAY`. Its `poll(..., 100)` is a maximum idle
wait, not 100 ms per packet: readiness/eventfd wake the loops. Its 8 KiB receive
limit is a maximum, not a fill requirement. UDP sends are nonblocking. These
facts rule out superficial diagnoses without proving negligible scheduling cost.

## DJI mechanisms and activation limits

Shared libraries contain slice-ready callbacks, frame/slice/packet end metadata,
bandwidth-derived encoder targets, QP/FPS adjustment, GDR recovery and late-frame
rejection paths. They support an architecture able to release work earlier and
manage picture age, but do not prove all those mechanisms run for O3.

| Retained evidence | Interpretation and limit |
| --- | --- |
| Slice-available events and low-latency append symbol | Available slice-aware path; not proof of slice decode or partial display |
| Explicit 6D frame/slice/packet end fields | Shared framing, including LTE/SDR branches; active O3 route untraced |
| Submit-delay warning above 3 ms; decode-stream warning above 7 ms | Diagnostic thresholds, not measured times or enforced deadlines |
| Display branch ignores delay above 20,000 µs | Branch-specific freshness check, not a glass-to-glass guarantee |
| Decoder hardware-start and frame-full callbacks | Separate events; start does not mean incomplete pixels are safe to display |

The goggles' Wi-Fi-sharing configuration selects its GDR settings, including
bandwidth feedback, `is_multi_slice_enc: 0` and `two_pass: 1`. These are not the
O3 air unit's proven encoder settings. The two-pass field's meaning was not
reconstructed; it does not establish offline multi-pass encoding or lookahead.

The WM1695 map includes source rates 29.984648/59.969296 fps mapped to 59.969 Hz,
49.87619516 fps to 99.75 Hz and 99.89131 fps to 99.892 Hz, all at 1920×1080.
Shared code names synchronized time, frame IDs, phase statistics and VSYNC
stop/start adjustment. Rate matching and phase alignment are distinct from
displaying an incomplete picture. The primary source config has
`vdec_disp_async_mode: false`; optional virtual-view defaults differ. Actual
negotiation and enabled modes remain unmeasured.

## Codec, bus and clock interpretations

Ascent's HEVC capture and DJI's HEVC default do not support a simple AVC-versus-
HEVC explanation. HEVC Main or `TRAIL_R` also does not establish B-picture
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

Compare stock local HDMI, proxy-enabled local HDMI, direct HEVC host playback
and integrated goggles separately. Match exposure, resolution, picture rate,
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

All 36 retained comparison-input hashes and 244 selected string offsets in nine
DJI binaries matched during this review. The parser probe and selected frame-
period, capacity and raw-pixel-rate calculations were checked again. No binary
control-flow reconstruction, radio operation, optical measurement or runtime
feature-activation test was added by those checks.
