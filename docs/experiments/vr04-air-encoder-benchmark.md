# VR04 reference air encoder benchmark

[Experiments](README.md) · [Media session protocol](../protocols/vr04-video-session.md)

## Setup and evidence

This consolidates the retained MissingLynk air benchmark at userspace revision
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`. It combines successive experiments;
its older tables and conclusions do not all describe the final settings.
Measurements are reported from those records, not rerun for this wiki. The
original packet capture and decoded pictures were not independently reanalyzed.

The test excludes the camera. Eight DMA-heap buffers are rendered in advance
and resubmitted to two concurrent HEVC instances. A 1920×1080 image becomes
1920×560 and 1920×552 tiles with 32 overlapping rows. This measures the encoder
and feeding path; it does not establish exposure, ISP or camera-to-screen latency.

| Content | Intended use and limitation |
| --- | --- |
| Static | Identical buffers; near-idle all-skip output after startup |
| Bars | Moving bars plus luma dither to retain residual content |
| Detail | Blurred noise moving differently across bands; its claimed worst-case role was not consistent across old settings |
| Noise | High-entropy planes; encoder survival stress, not a realistic RF bitrate target |
| Scroll | Render each frame from a shared counter; suitable for visual inspection, but rendering limits its frame rate |

The pre-rendered ring reuses buffers in completion order, and tile phases are
independent. Apparent motion jumps or seams in those tiers can come from the
generator itself. Scroll avoids that particular artifact; it is a separate test.

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
| `tx` | Successful local socket send, not remote receipt |
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

Remaining measurements include a higher-throughput harness, maximum-rate detail
and noise, 10 Mb/s per tile, deliberately exercised overflow recovery, controlled
camera scenes and synchronized receiver results under load. The recorded 46 ms
ping RTT does not measure the video queue. Receiver-local timing is documented
separately in the [latency study](vr04-receiver-latency.md).
