# VR04 reference air encoder benchmark

[Experiments](README.md) · [Media session protocol](../protocols/vr04-video-session.md)

## Setup and evidence

This consolidates the retained MissingLynk air benchmark at userspace revision
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`. It combines successive experiments;
its older tables and conclusions do not all describe the final settings.
Measurements are reported from those records, not rerun for this wiki. The
original packet capture and decoded pictures were not independently reanalyzed.

The test excludes the camera. The default ring requests eight DMA-heap buffers
per tile, rendered in advance and resubmitted to two concurrent HEVC instances.
Allocation can yield a smaller pool. A 1920×1080 image becomes
1920×560 and 1920×552 tiles with 32 overlapping rows. This measures the encoder
and feeding path; it does not establish exposure, ISP or camera-to-screen latency.

| Content | Intended use and limitation |
| --- | --- |
| Static | Identical buffers; near-idle all-skip output after startup |
| Bars | Moving bars plus luma dither to retain residual content |
| Detail | Blurred noise moving differently across bands; its claimed worst-case role was not consistent across old settings |
| Noise | Deterministic pseudo-random planes with 128 possible byte values; encoder survival stress, not a realistic RF bitrate target |
| Scroll | Render each frame from a shared counter; suitable for visual inspection, but rendering limits its frame rate |

The pre-rendered ring reuses buffers in completion order, and tile phases are
independent. Apparent motion jumps or seams in those tiers can come from the
generator itself. Scroll avoids that particular artifact; it is a separate test.

The noise renderer uses `16 + (byte & 0xdf)`, yielding 128 levels within
16–239. It is not full 8-bit entropy, and luma can exceed the declared studio
maximum of 235. The detail renderer uses a deterministic 3×3 box-blurred texture,
eight moving luma bands and lower-amplitude chroma. Neither generator establishes
a universal camera-content worst case. An unrecognized tier name silently uses
the bars renderer, while the log retains the supplied name.

## Later 60 fps encode-only result

The later table uses **5 Mb/s per tile, minimum QP 25, initial I QP 32,
CU/MB rate control enabled, VBV 78 ms and GOP 0**. Each tier runs for 15 seconds
within one instance pair and one boot. The recorded GOP 65535 in older setup
prose was outside the accepted 0–2047 control range; it did not establish that
interval on hardware. The report says the default 0 applied instead.

| Tier | Completed frames, tiles 0 / 1 | Encoded bitrate per tile | Largest access units, bytes | Below 65,467 bytes? |
| --- | ---: | ---: | ---: | --- |
| Static | 896 / 896 | 0.06 Mb/s | 422 / 450 | Yes |
| Bars | 863 / 863 | 0.30 Mb/s | 803 / 764 | Yes |
| Detail | 865 / 865 | 5.01 Mb/s | 44,572 / 44,334 | Yes |
| Noise | 899 / 898 | 37.0 Mb/s | 77,487 / 76,622 | No |

The report records zero ring drops, lost outputs and `VLC_BUF_FULL` errors.
Its “held 60 fps” describes the steady paced portion: rewriting the ring between
tiers is inside the measurement window, so the full-window counts do not all
equal 900 or average exactly 60 fps.

Enabling the aggregate CU/MB rate-control setting reduced the reported detail
peak from 276,948 to 44,572 bytes, about 6.2 times; noise fell from 656,422 to
77,487 bytes, about 8.5 times. Thus the old conclusion that detail could not fit
one datagram is superseded for the later configuration. Noise still exceeded
the ceiling by about 18%, and its 37 Mb/s per tile output was not a successful
RF-transmission result.

The old minimum QP of 8 also explains the observed easy-content overshoot:
raising it to 25 reduced bars from roughly 5.9–7.3 Mb/s to 0.30 Mb/s. A universal
“CBR is always 1.6 times the target” correction is unsupported. These benchmark
values are distinct from the later air default minimum QP of 15.

## Vendor capture comparison

The retained vendor capture summary contains 2017 video packets: two IDRs and
2015 P pictures across the same tile geometry.

| Statistic | Reported bytes |
| --- | ---: |
| Maximum IDR / overall maximum | 18,919 |
| Mean / maximum P picture | 11,154 / 18,174 |
| p50 / p99 / p99.9 | 12,029 / 16,798 / 18,040 |
| Pictures above 65,467 bytes | 0 |

An earlier 18,000-byte “ceiling” was a histogram bucket, not a discovered encoder
limit. The vendor maximum was about 1.7 times its mean P-picture size. Later
synthetic detail peaked around 4.3 times its own mean; the source's retained
34-times comparison describes an older result.

Real camera content and synthetic detail are different workloads. One comparison
also labels 5.35 versus 4.0 Mb/s as matched bitrate, which they are not. These
observations support a flatter captured vendor distribution, but do not isolate
one QP setting as the complete cause or guarantee that all camera scenes fit.

## Air configuration and parity work

The stock air encoder's fallback configuration comes from `stVencTxMap`, a table
of twenty 64-byte entries used when the persisted encoder configuration is absent.
Its values must be distinguished from goggle RTSP channel 8.

| Air setting | Recorded default | Accepted range |
| --- | ---: | --- |
| Enable / encoding type | 1 / 2, HEVC | 0–1 / 0–5 |
| GOP / rate-control mode | 0 / 0, CBR | 0–200 / 0–6 |
| Initial configured bitrate | 2000 kb/s | 128–16,000 kb/s |
| Key / non-key buffer multiplier | 4 / 2 | 1–100 each |
| Minimum / maximum I and P QP | 15 / 51 | 0–51 |
| Statistics time | 0 | 0–60 |
| Skip threshold | 5 | 0–10 |
| Automatic ROI threshold | 682 | 100–10,000 |

Live bitrate derives from RF throughput/MCS, so the configuration's 2000 kb/s
seed is not a measurement of the running stream. The
[link-control study](../systems/vr04-link-control.md) covers the 0.7 throughput
factor, 20 Mb/s total cap, zero-throughput fallback and open implementation's
additional menu cap.

The service's lower-level template is built by
`ar_video_h26x_enc_exe_gen_default_param` at `0xa59b8`, from a non-contiguously
copied block at `0x304f20`–`0x30506f` into the channel object near `+0x1e90`.
Following named log loads and copy order matters; reading the template as one
flat structure gives incorrect fields. Independent anchors were noise weights
7/7/7 and 4/4/4, initial RC QP −1 and first intra QP 35, also reported on the wire.

Six differences were subsequently implemented in the open driver:

| Parameter | Earlier open value | Later value matching the recovered template |
| --- | ---: | ---: |
| Maximum HVS QP delta | 10 | 4 |
| Buffer rate-control weight | 128 | 1 |
| RDO skip | 1 | 0 |
| Lambda scaling | 1 | 0 |
| Temporal-MV prediction | 0 | 1 |
| Maximum merge count | 0 | 2 |

**None of the benchmark measurements in this source used those six later
changes.** Their implementation does not supply a new measured parity result.
The driver retains HVS scale 2; a public RC-structure interpretation of 4 remains
unreconciled. The template's intra-QP offset −5 had no corresponding open-driver
field/control in this record. Goggle RTSP's QP floor 0 is a separate setting from
air floor 15, and both differ from the benchmark's 25.

The I-frame proportion fields were stored and returned but no consuming path was
found. They therefore do not explain the flatter vendor packets. Buffer multipliers
also allocate capacity rather than impose a picture-size target:
`align16(multiplier × buffer_size / 8)`, capped at raw frame size. At multiplier
4 and 1920×560 4:2:0, that gives 806,400 bytes, not a roughly 19 KB rate limit.
The [stock MPP study](../systems/vr04-stock-media.md) retains the API layout gaps.

## Packet budget and counter interpretation

The open packetizer permits a 65,507-byte UDP payload. Subtracting the 36-byte
VPH header and four-byte trailer leaves **65,467 bytes for HEVC including inserted
SEI**. This is the one-datagram open path, distinct from other products' splitting
policies described in the [session protocol](../protocols/vr04-video-session.md).

| Counter | What it establishes |
| --- | --- |
| `pushed` | Input submitted to the encoder |
| `done` | Encoder output returned, independent of transmission |
| `dropped` | Ring-slot starvation; attribution can fall on the stalled tile's partner |
| `tx` | Successful local socket send in transmitting mode; also increments with `ML_AIR_NOTX`, where no send occurs |
| `oversize` | Locally rejected oversized output in a transmitting run |
| `txerr` | Local send error |
| Largest AU | Output-size diagnostic, including encode-only tests where `oversize=0` says nothing about fit |

At fixed 4 Mb/s, mean bytes per picture are roughly 8333 at 60 fps, 16,667 at
30 fps and 33,333 at 15 fps. Means do not bound peaks. One 15 fps run reported
tile 1 producing 15 but transmitting 12 pictures/s, about 20% discarded before
the socket. Both netdev drop counters and UDP send-buffer errors stayed zero.
This explains that incident, not a rule that every sub-30-fps stream fails.

The older claim that VBV sets a hard per-picture ceiling was contradicted by
later measurements. At 4 Mb/s, 98 ms corresponds to a nominal 49,000-byte budget,
yet earlier detail/noise peaks reached roughly 277/656 KB. The searched channel
attributes, configuration table, RC parameters, multiplier logic and driver
register path revealed no hard byte-cap control. That is a bounded search result,
not proof that every possible encoder configuration or firmware lacks one.

## What the air latency instrumentation measures

The current `mav-latstats.c` instrumentation at the revision above describes the
direct camera/encoder path. It does not add latency measurements to the historical
synthetic benchmark tables. Presence of `ML_AIR_LATSTATS` enables summaries;
`ML_AIR_LATRAW` enables both raw records and summaries. A value of `0` still enables
either option.

| Logged interval | Actual endpoints |
| --- | --- |
| `capts2tx` | Capture-buffer timestamp to userspace time after a successful local datagram send |
| `capdq2tx` | Userspace time after capture dequeue to that send-return mark |
| `dq2q` | Capture dequeue mark to userspace time after successful encoder input queueing |
| `q2done` | Input-queue return mark to userspace time after coded-output dequeue |
| `done2tx` | Coded-output dequeue mark to send-return mark, including intervening packet preparation |
| `txskew` | Absolute difference between the two tiles' send-return marks for one capture ordinal |

With `ML_AIR_NOTX`, the final mark is taken without calling the socket, and `sent`
still advances. Neither mode measures RF delivery, receiver decoding or screen
presentation. The capture timestamp is copied from the V4L2 buffer; the other
marks use the userspace monotonic clock. Their subtraction assumes compatible
clock domains. The [CVISP review](../soc/ar9311-nt99235-camera.md#capture-metadata-and-diagnostic-controls)
identifies capture-completion timing, rather than an optical exposure-start mark.

The capture identity used for pairing is the program's incrementing dequeue
count, narrowed to 32 bits; it is not the driver's `b.sequence`. Each tile queues
traces in a FIFO after successful encoder input submission, then removes its
oldest trace when it accepts a nonempty mapped coded output. It does not match
that output through a returned input timestamp or buffer identity. Attribution
therefore assumes one accepted output per input in order. A full trace FIFO
omits new traces, while an empty or invalid coded output does not remove one;
the bookkeeping alone does not establish correct attribution after such cases.

Only records with all five timestamps positive enter the accumulator. In
transmitting mode, failed sends, oversized datagrams and outputs withheld by
the startup gate do not reach this timing call. Thus these are statistics for
eligible successful local sends, not every captured or encoded frame. A null
trace also contributes nothing.

Each tile retains its first 256 eligible samples until the next flush, normally
requested by a one-second main-loop timer. A delayed timer changes the window;
the code does not normalize by actual elapsed time. Durations are rounded to
0.1 ms **before** sorting. The percentile selects zero-based index
`floor((n - 1) × p / 100)`: with 60 samples, p99 selects the second-largest,
unlike the receiver's nearest-rank statistic. An empty population prints zero.
Positive timestamps are checked, but their ordering is not: a negative interval
becomes the sentinel `-1`, participates in sorting and prints as **−0.1 ms**.
This is not a physically measured negative latency.

Pair skew has its own 256-sample cap and a 64-entry ring keyed by capture ordinal.
A collision replaces an unfinished pair. Completed pairs are cleared, so a
repeated pair of records for the same ordinal can count again. Summary flush
clears the accumulators but retains unfinished pairs: a pair can complete across
two reporting windows. Consequently, the skew population need not equal either
tile's printed sample count or belong wholly to that interval.

Offline checks of the unchanged accumulator implementation, with host type and
logging adapters, reproduced the percentile, negative-interval, sample-cap,
ring-collision, duplicate-pair and cross-window behavior. These checks exercised
no encoder, camera, socket or RF link, and did not reprocess historical captures.

## Transmission and picture validation are separate runs

A separate 60 fps transmitting boot reported 896/896 static frames and 858/858
detail frames, with `tx=done`, zero reported local drops and approximately
3.9–4.1 Mb/s per tile for detail. The interface sent 14,361,959 bytes in 7042
packets without reported errors/drops. The host could not inspect the goggle in
that run, so it did not verify reception, decoding or display. Older unsent
bars/noise bitrate estimates must not be substituted for the later QP-25 table.

The later visual run used **300 seconds, 15 fps requested, 1 Mb/s per tile,
GOP 30 and VBV 392 ms**. Scroll bars were reported smooth on the panel with no
visible tile seam. Rendering sustained about 13 fps, with `tx=done`, zero oversize
or transmit errors, peaks of 422/450 bytes and only 0.01 Mb/s per tile. This
validates a low-load picture/transport session; it does not validate the demanding
encode-only tiers or full-rate RF reliability. It supersedes the old blanket
statement that no picture in the study had ever been inspected.

## Throughput ceiling and failure recovery

Earlier separate-boot maximum tests reached 126 fps for static and 114/115 fps
for bars. Near-idle content was only about 10% faster, suggesting the feeding
harness itself limits throughput near 126 fps. The roughly 1.9-times margin over
60 fps is a result for that complete benchmark path, not a measured encoder IP
ceiling. Decoding dumped output into 476 frames of the expected geometry checked
frame count and shape; it did not inspect every reconstructed pixel.

Historical overflow returned `-5`, reason `0x10000` (`WAVE5_SYSERR_VLC_BUF_FULL`),
then stopped picture submission while retaining the ring buffers. Later noise
ran 15 seconds at 60 fps without reproducing the error, so that run **did not
exercise overflow recovery**. A corrected second coded-buffer `sizeimage` clamp
was a proposed explanation for the improvement, not an isolated causal test.
Output-buffer capacity and transport packet budget are separate limits.

The old requirements for one parameter point per boot, no hot reload and no
on-demand IDR predate the later reset/reload and live-parameter implementation.
See [codec recovery validation](vr04-media-validation.md) for those changes and
their remaining reuse/recovery limits. Their existence does not retroactively
turn this overflow-free benchmark into a successful fault-injection test.
The later [air capture and encoder implementation review](../systems/vr04-air-capture.md)
also distinguishes buffer-credit release from DMA quiescence, partial live
updates from transactions, and tile retirement from whole-pipeline recovery.

## Current runner and result interpretation

Complete review of `ml-air-video.c`, `mav-bufs.c` and `mav-bench.c` at the pinned
revision adds these limits to the historical reports. This is not a reanalysis
of their original captures or a new throughput measurement.

The benchmark uses the **GStreamer** encoder path. Its encoder element and
properties come from `ML_AIR_ENC`; the program's default still requests GOP
65535. The direct-camera initialization's bitrate, VBV and QP setters do not run
for this path. Reproducing the recorded parameter set therefore requires the
actual GStreamer encoder configuration, rather than assuming that direct-camera
`ML_AIR_BITRATE`/QP environment variables configure a benchmark run. The selected
mode also matters: setting both `ML_AIR_CAMERA` and `ML_AIR_BENCH` is not rejected,
but can select the benchmark feeder while skipping GStreamer encoder creation.
It is not a supported combined camera/benchmark mode.

The pool requests 2–16 buffers per tile and stops on allocation or mapping
failure; startup requires at least two. Heap selection first tries a named heap,
then the first openable non-`mmz` entry, then any openable heap. An allocation
failure on the selected heap does not try another heap. Thus the retained
comment that every pool consumes the same 32 MiB `mmz` region is platform-specific,
not the allocator's universal policy. Normal tile allocations are each
1,612,800 bytes: four per tile request 12,902,400 bytes in total, while eight per
tile request 25,804,800 bytes, before allocator overhead or encoder working memory.

Content is packed at the coded height; allocation padding is at the tail.
Each submission still creates a buffer wrapper, metadata and three duplicated
FD-backed plane views per tile. Pre-rendering avoids repeated pixel generation;
it does not remove all per-frame allocation or FD work. CPU-write sync ioctls
are unchecked, so renderer completion does not prove DMA visibility.

The first GStreamer frame normally submits tile 1 then tile 0, waiting up to
300 ten-millisecond sleeps for each tile's first output. A failed push or wait
timeout still leads to the other tile and ultimately sets `g_primed`. Later
frames do not retry this startup gate. The waits are inside the first tier's
wall-time budget. `ML_AIR_ORDER=01` reverses this helper's order; this is separate
from direct-camera startup.

Later tiers rewrite each pool slot when it becomes free; there is no full drain
or content-change barrier. The tier timer starts before those rewrites and before
old outputs necessarily finish. A short tier need not renew every slot, and
outputs from the previous tier can be counted under the next label. `scroll`
instead renders undithered bars per accepted frame from the shared counter.
The counter advances only after a successful pair reservation and remains
continuous across tiers; its synthetic PTS is not a measured capture timestamp.

| Runner output | Scope of the number |
| --- | --- |
| Per-tier `frames` | Difference in completed-output counter since tier entry; no boundary drain |
| Per-tier `oversize`, `tx errors`, `lost`, `largest AU` | Cumulative process counters/high-water mark, not reset at tier entry |
| Periodic `/s` and `Mbit/s` | Counter deltas per nominal one-second callback, without division by actual elapsed time |
| `STALLED` / `FAIL` | More than two seconds since the last **observed output from either active tile** |

A healthy tile can keep the shared stall timer fresh while its partner produces
no frames. Conversely, unpaced mode can skip progress observation when waiting
for a free buffer fails, so the timer is not a direct hardware completion clock.
The verdict is logged at tier end; it neither stops subsequent tiers nor changes
the process exit status. GStreamer ERROR/EOS also quits the main loop without
setting a failure exit code: after ordinary cleanup, `main` returns zero.
Early setup failures do return nonzero. A successful shell exit alone cannot
establish benchmark acceptance.

Offline checks of the exact renderer and selected feeder/pool functions reproduced
pair-reservation rollback, priming after failed startup pushes, one progressing
tile masking its stalled partner, cumulative diagnostic fields, and the noise
renderer’s 128-level distribution and untouched allocation tail. They passed
AddressSanitizer and UndefinedBehaviorSanitizer checks with mocked queues,
clock and encoder pushes. No GStreamer negotiation, DMA or physical rate was tested.

Remaining measurements include a higher-throughput harness, maximum-rate detail
and noise, 10 Mb/s per tile, deliberately exercised overflow recovery, controlled
camera scenes and synchronized receiver results under load. The recorded 46 ms
ping RTT does not measure the video queue. Receiver-local timing is documented
separately in the [latency study](vr04-receiver-latency.md).
