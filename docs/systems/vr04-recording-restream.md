# VR04 recording, subtitles and RTSP restreaming

[Systems](README.md) · [Open media overview](vr04-open-media.md) ·
[Playback](vr04-playback.md)

## Scope and shared encoder

This study covers the complete recording and RTSP modules at MissingLynk
revision `674ec00ecde0bba2eb844def1b4a0b7f8e685e13`, together with the HUD command
wrapper and selected supporting declarations. It is a source review with
bounded offline checks, not a new storage, decoder, network-client or hardware
qualification. The [media experiments](../experiments/vr04-media-validation.md)
retain the measured results and encoder-reuse failures.

One encoder consumes completed receiver composites. Its encoded output splits
into an optional MP4 branch and an always-present appsink branch feeding RTSP.
With RTSP enabled and no recording, it runs without a file branch. H.264 is the
default; `ML_DVR_CODEC=h265` or `hevc`, case-insensitively, selects HEVC. The
RTSP module's older opening comment saying HEVC is the default is superseded
by its actual codec selection.

The normal format table contains 1920×1080 and 1280×720, each at 30 or 60 fps.
HUD settings are latched for the next encoder start. Bench environment values
override them: unsupported heights select native resolution, and frame rates
other than 30 resolve to 60. Environment switches disabling hardware scaling or
DMA import are tested by presence, so setting their text to `0` still disables
the feature.

## Scaling, queues and buffer ownership

Native import wraps one composite DMA-buffer descriptor as three plane-memory
objects and retains the original composite through buffer-attached data. The
allocation includes encoder size-image padding while the planes retain their
packed offsets. The wrapper avoids a full pixel copy; its lifetime still pins
a composite pool slot until the encoder releases it.

720p hardware scaling uses a persistent pool of up to four destination buffers,
with at least two required. A worker consumes a one-frame, newest-wins mailbox
so the scaler call does not block the decoder callback. The source composite is
released after the scale ioctl; a finalization hook returns the destination slot
after the encoder releases its wrapper. If all destinations are busy, the worker
drops the frame. Failed scaler calls return the slot and increment a separate
failure counter.

For 720p30, a PTS gate requiring at least 25 ms since the prior admitted timestamp
runs before scaling. This selects alternate frames from ideal 60 fps input but
is not an independent wall-clock scheduler. Native 30 fps instead uses
`videorate`. A menu request for scaling falls back to native when the hardware
path is unavailable; explicit bench overrides can still select the older CPU
scaling path with its retained zero-output failure history.

| Input mode | Configured appsrc buffer limit |
| --- | ---: |
| Native DMA import | 3 |
| Copy path | 6 |
| Hardware scaling | `max(1, min(pool count − 2, 3))` |

The recording input checks appsrc depth and also configures a leaky downstream
queue. These limits do not count encoder-internal references, the scale worker
or its mailbox. The native copy path takes a reference to the existing buffer
and changes its timestamp metadata; it does not first make an independent
metadata wrapper as the import path does.

`pushed` counts successful appsrc submissions, not encoded pictures or durable
file writes. `dropped` is not a complete frame-loss total: 30 fps selection,
scaler failures, downstream leaky queues and an abandoned mailbox at stop are
not all counted there. The comments' steady-state scaler timing does not prove
that the mailbox can never overflow under other workloads.

## Start, file naming and finalization

The default GOP request is 60 frames. That is nominally one second at 60 fps
and two seconds at 30 fps. A positive bitrate override enables frame-level rate
control; otherwise this wrapper leaves the fixed-QP default. Neither the
configuration string nor its log establishes a hard instantaneous byte-rate
bound. The [codec review](../experiments/vr04-media-validation.md#parameter-changes-and-keyframes)
describes the driver-side keyframe behavior separately.

Recording state becomes true after the bin accepts a PLAYING request and, when
needed, the scale worker starts. There is no first-encoded-frame or nonzero-file
gate. Later bus errors schedule a stop on the main loop. A recording indicator
can therefore precede output or coexist with a silent encoder stall.

Automatic naming scans the selected directory for the largest parsed
`Video<number>.mp4` and chooses the next number. It does not reserve the name
atomically or require a mounted card or regular file. A failed directory scan
starts again at `Video001.mp4`. A pinned recording path can overwrite an
existing recording. The path is interpolated unquoted into the GStreamer
description, so arbitrary filenames containing pipeline syntax or spaces are
not covered by the normal generated-name behavior. A stale SRT at the selected
path is removed before successful encoder startup is known.

The MP4 muxer requests 1000 ms fragments. This is intended to retain completed
fragments after an interrupted process, but this source alone does not establish
a power-loss guarantee or a strict one-second loss bound. Fragment production,
keyframe spacing and filesystem persistence are distinct conditions.

Stop first clears software recording/encoder state, joins the scale worker and
closes subtitles. It removes the bus watch, sends appsrc EOS and waits up to
three seconds for either EOS or ERROR. It does not distinguish those messages,
report a wait timeout or require successful EOS before requesting NULL and
releasing the bin. “Stopped” consequently does not certify a finalized MP4.
The worker join and state-change calls also mean the three-second bus wait is
not an overall stop deadline.

Toggling recording while restreaming rebuilds the encoder with or without the
file branch. The RTSP server remains, but this introduces a media gap; continuity
depends on subsequent output and decoder entry points. Failed recording start
can be followed by a separate attempt to restore the stream-only encoder.

## Subtitle timing and stalled video

SRT creation is lazy: the first nonempty subtitle command while recording opens
the sidecar. Text is retained in a 256-byte pending buffer and written when the
next text arrives or recording stops. The timestamps use recording-relative
input PTS, not subtitle arrival wall time. `rec_last_ms` is updated before the
encoder appsrc push result is known, so it can describe an attempted frame that
was not accepted, rather than an encoded frame present in the MP4.

Ordinarily the next cue starts where the preceding cue ends. When video time
has not advanced, however, the writer extends the old cue to one second after
its start while keeping the new cue's original start. This produces overlaps.
An offline check yielded a first cue at 1000–2000 ms and a second at
1000–1500 ms. The sidecar is therefore not unconditionally gapless or
nonoverlapping during stalls. Final cue time also does not establish the
duration of the last encoded picture, and write/close results are not checked.

## RTSP lifecycle and timestamps

The server mounts `/venc8/stream` on port 554 by default, with a shared media
factory using the selected encoder codec. A port override is available for
bench use. Failed bind attempts have a 30-second retry delay. Session-pool
cleanup runs every two seconds; that interval is not itself a client timeout.

Enable commands retry missing server/encoder setup. Disabling closes existing
clients and stops a stream-only encoder, but keeps the server and mount attached.
It does not unbind the listening port. The RTSP state flag is the conjunction
of enable intent and the encoder-start flag: it does not verify a prepared
client, encoded bytes, successful RTP delivery or a visible picture.

Each newly configured shared media object waits for a keyframe and requests one
from the encoder. The upstream request's return value is ignored. A delayed
unprepare callback checks that its appsrc is still the current one before
clearing it, protecting a newer feed from that stale callback.

The restream clears copied access units' PTS/DTS and lets its appsrc timestamp
them against the media's running time. This avoids recording-bin timestamps
resetting to zero inside an existing RTSP session. It also means RTSP timestamps
are not the original RF arrival, capture time or MP4 timeline.

## Parameter sets and loss recovery

Parameter-set NALs are cached even while restreaming is disabled. There is one
slot per kind—HEVC VPS/SPS/PPS or H.264 SPS/PPS—and the concatenated cache is
rebuilt when a kind appears. It is not indexed by parameter-set ID, checked for
a complete compatible set, or explicitly cleared on every encoder-bin rebuild.
Partial updates can retain kinds from an earlier encoder configuration.

The parser stops at the first VCL NAL and uses it to classify an H.264 IDR or
HEVC IRAP. A leading parameter-set prefix of **any nonzero length** suppresses
injection of the entire cache. Thus an SPS-plus-IDR unit without PPS is treated
as carrying its own prefix, even if a cached PPS exists. The offline check
confirmed this decision; it did not test whether a particular client could
decode using previously retained state.

The keyframe gate is cleared before sending the cached prefix and picture.
The prefix push result is ignored, and a failed picture push increments a
counter without re-arming the gate. A following non-key unit can therefore be
attempted after the entry point was rejected.

The encoder tee's RTSP queue can drop older encoded units, and the server appsrc
has a separate 16-buffer leaky queue. Neither loss path explicitly closes this
gate or immediately requests a new keyframe. Periodic entry points can restore
decodability, but the comments' “self-heals at the next IRAP” is conditional on
such an entry point being emitted, delivered and accepted. The feed counter
tracks appsrc acceptance, not client delivery. These queues protect upstream
progress by permitting loss; they do not prove uninterrupted restream playback.

## Offline checks and remaining limits

Exact extracted subtitle and RTSP feed functions were compiled with mocked
GStreamer services. A separate check used the complete control-dispatch function,
the actual wire header and an anonymous Unix datagram socket pair. Both passed
C11 warnings-as-errors and AddressSanitizer/UndefinedBehaviorSanitizer checks.

The checks reproduced subtitle overlap, partial-prefix suppression, and a key
gate remaining open after failed pushes. They also measured the
[maximum-size OSD command truncation](../protocols/mlm-ipc.md#pipeline-commands-and-state)
and showed that the receiver itself does not enforce the sender helper's
512-byte path limit. They did not exercise GStreamer scheduling, buffer-import
ABI, external RTSP clients, hardware scaling/encoding, client decoding, storage failure
or MP4 finalization.
