# VR04 air capture, encoder ownership and recovery

[Systems](README.md) · [Camera/ISP](../soc/ar9311-nt99235-camera.md) ·
[Air benchmark and timing](../experiments/vr04-air-encoder-benchmark.md) ·
[Media session](../protocols/vr04-video-session.md)

This reviews the air transmitter at MissingLynk userspace revision
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`. Complete implementation review covers
capture, direct encoding, live controls and their client, startup/shutdown,
buffer pools and the benchmark runner.
It describes the VR04 reference camera path, not the SC231AI P1 Air pipeline or
a new hardware qualification. Historical encoder tests and kernel recovery
results remain separate evidence.

## Capture geometry and buffer ownership

With a camera selected, the default path uses direct V4L2 encoder ioctls. Presence
of `ML_AIR_GST`, even with value `0`, selects the GStreamer alternative. The direct
path explicitly supplies the capture strides when configuring encoder input;
older comments saying per-buffer GStreamer video metadata is sufficient must
not be read as validation of that alternative.

The top-level comment also describes tile-1-first staggered startup as if it
applied throughout. That wait-for-first-output helper is used by the GStreamer
path. Default direct-camera startup and deferred startup iterate tile 0 then
tile 1, without waiting for a first coded frame between them. `ML_AIR_ORDER`
and `ML_AIR_NO_STAGGER` do not change those direct open loops. The historical
staggered-start result therefore does not validate every current startup path.

| Resource | Current policy |
| --- | --- |
| Capture buffers | Request defaults to 8, clamped to 2–16 before allocation; returned count capped at 16 |
| Capture frames held by encoders | Default limit 3; `ML_AIR_INFLIGHT` has a lower clamp of 1 but no upper clamp against the allocated capture count |
| Encoder input slots | Request 8 DMABUF slots per tile; capture buffers own the memory |
| Encoder coded-output buffers | Request 6 MMAP buffers per tile |
| Capture exports | Three planes per buffer, wrapped once as `GstMemory`; duplicate FDs serve direct queueing |
| Normal tiles | 1920×560 at row 0 and 1920×552 at row 528, with 32 rows of overlap |

The tile's input extent uses height rounded up to 16 rows. For the second tile,
row 528 plus 560 allocated rows reaches row 1088, while its coded height remains
552. Offsets and lengths use the driver's individual plane strides. This source
allocation layout is distinct from the receiver's packed decoded-image layout.

Capture startup reads the existing format and requires 1920×1080 with three
planes. It does not independently check the pixel-format code or minimum plane
strides. Tile bounds, and required CPU mappings for copy mode, are checked
against capture buffer **zero**, rather than every exported buffer. Initial
capture `QBUF` failures are logged but do not themselves abort startup. These
checks rely on the matched driver's layout and allocation behavior.

Direct encoder startup rejects a substituted input format/plane count, a changed
stride, or a reported plane size larger than the tile's supplied extent. It does
not independently compare the returned input width/height, verify the coded
format returned by the second `S_FMT`, or bound returned queue counts against
the fixed 8/6-element arrays. Coded-output dequeue checks a mapped buffer index
and nonzero `bytesused`, but does not check
`data_offset <= bytesused <= mapped length` before passing the span onward.
These are driver-trust boundaries, not demonstrated failures of the retained
driver.

For each accepted direct-path capture frame, the feeder increments the global
in-flight count once and sets the buffer reference count to the number of active
tiles. Each successful encoder queue owns one reference; a failed queue releases
its share immediately. Returning an encoder **input** buffer releases that
reference independently of coded-output dequeue. The final release decrements
in-flight credit and attempts to return the capture buffer to the camera.
Failed camera requeue only logs an error: credit has already been returned,
so the counter does not prove the camera rotation recovered that buffer.

The GStreamer path instead attaches finalizers to shared input views. Copy mode
reserves tile-pool slots before copying, and a reservation failure returns all
reserved slots and the camera frame. Its starvation counter is charged to tile
0 even if the other tile lacked a slot. The direct-path branch precedes the
copy/share branch and ignores `tile->copy`; setting `ML_AIR_COPY` alone can
allocate a pool without making the default direct encoder use copied input.

## Rate selection and on-demand startup

Frame selection uses the capture timestamp and integer-truncated frame period.
Frames earlier than the next due time, or with no in-flight credit, are returned
unused. `capfps 0` takes every available frame; changing this feeder setting
does not change encoder rate control. It is a decimation policy, not a request
to change the sensor's frame rate.

Presence of `ML_AIR_ON_DEMAND` defers direct encoder opening until a keyframe
request. Camera streaming starts first and continues while encoders are held.
The feeder services the open request before polling, opens all active tiles, and
closes them all if any open fails. A successful open sets `g_enc_up`; it does not
establish that a usable IDR was emitted, sent or decoded.

The requester waits on a condition variable for up to five seconds. Timing out
does **not** cancel the pending open: `g_enc_start_want` remains set until the
feeder services it. The feeder's device operations have no corresponding
userspace deadline. A client can therefore receive an error while a bring-up
continues or subsequently succeeds. Once already up, a keyframe command attempts
the force-keyframe control on each active encoder; this handler has no per-frame
spacing gate of its own. Kernel behavior and the separate transmitter retry
gate must not be inferred from its `ok` reply.

## Live controls are not transactions

| Command | Ordering and limits |
| --- | --- |
| `bitrate <bps> [vbv]` | 1–700,000,000 bps per tile and 10–3000 ms VBV; bitrate then VBV on each tile |
| `fps <fps>` | 1–240; feeder changes first, then each encoder's `S_PARM` |
| `rate <bps> <fps> [vbv]` | Validate FPS, apply bitrate/VBV, then FPS only if that first operation succeeds |
| `capfps <fps>` | 0–240; update feeder alone |
| `keyframe` | Start deferred encoders, or attempt a force-keyframe control on each open active tile |
| `session-reset` | Flag a reset for adoption by later access-unit processing |

Omitted or negative VBV uses a heuristic derived from three quarters of the
65,467-byte access-unit budget, clamped to 10–3000 ms. At 4 Mb/s this yields
98 ms. It is not a hard per-picture byte limit; the
[benchmark](../experiments/vr04-air-encoder-benchmark.md#packet-budget-and-counter-interpretation)
documents peaks beyond the nominal budget.

Startup integer-control failures are logged and ignored. A successful open
caches the requested bitrate/VBV, and the SEI QP field describes the configured
value, not a measured per-picture QP. Later setters skip values equal to that
cache. Thus an initially rejected setting may not be retried by a same-value
live command. Live setter success updates its cache, but does not read the
setting back or prove firmware application.

Updates continue after individual bitrate/VBV failures and have no rollback.
For example, tile 0 can adopt a new bitrate while tile 1 rejects it, yet both
adopt the new VBV. An FPS failure can similarly leave the feeder and tile 0 at
30 fps while tile 1 retains 60 fps. Returning `err` reports the failed operation;
it does not mean nothing changed. Before deferred encoders open, bitrate/FPS
commands reject the request because no active open encoder exists; they do not
save a desired configuration for that later open.

The Unix stream control handler performs one read of at most 255 bytes after a
100 ms readiness poll. It does not accumulate a complete line across fragmented
stream reads. Numeric parsing accepts trailing text, while `keyframe` and
`session-reset` use prefix matches. Reply writes are unchecked. The 100 ms wait
is not a whole-command timeout: startup and device controls can take longer.

The `ml-air-ctl` client has no internal connect/read deadline. It formats at most
127 command bytes without rejecting truncation, performs one write and one
reply read, and returns success if that read begins with `ok `. It does not
accumulate a fragmented reply or query applied state. Its exit status therefore
reports the control exchange's result, not an observed encoder or video result.

## Stall detection and retirement

The feeder drains encoder input/output readiness before checking capture credit.
Encoder `POLLERR` or `POLLNVAL` requests recovery. The silence check requires
held input buffers, more than 500 ms since the encoder last returned a buffer,
and less than 500 ms since the camera last supplied one. Returning either input
or coded output counts as progress, so this is not specifically a missing-video
watchdog. A poll timeout/error skips the checks for that iteration; a quiet
camera also suppresses silence recovery.

Recovery closes and reopens the encoder instance, despite an older feeder
comment describing an OUTPUT-queue cycle. The first five recovery calls can
attempt a reopen. The next call retires the tile; an earlier failed reopen also
retires it immediately. The counter is cumulative, not reset after successful
frames. The other tile can continue, but this does not guarantee a complete
receiver picture. Reopening does not reset the transmitter's existing frame
sequence or per-tile IDR-sent state.

Close attempts `STREAMOFF` on both queues, ignores both results, releases held
capture references, unmaps coded buffers, then closes the FD. The release can
requeue a camera buffer **before** the encoder FD is closed even if both
`STREAMOFF` calls failed. Software reference release is therefore not evidence
of DMA quiescence; that guarantee depends on the matched kernel's teardown.
Normal direct-path shutdown joins the feeder, closes encoders, then closes
capture. The exported capture memories are deliberately retained until process
exit; this is not a reusable capture-close/free operation.

If recovery retires every tile, it leaves `g_enc_up` set. The direct feeder
counts zero active tiles but still increments in-flight credit and sets the
capture reference count to zero. No tile can perform the final release, so
those buffers are withheld until the credit gate stops admitting more. A
startup request can return success from the stale up flag, while a keyframe
command rejects the absence of active encoders. Retirement therefore contains
repeated encoder reopen attempts but does not implement recovery of the whole
pipeline after every tile is lost.

GStreamer ERROR/EOS callbacks request main-loop exit but do not set a nonzero
process result. After ordinary cleanup the program returns zero. Control-socket
open failure is also ignored by the top-level caller. Neither a successful
process exit nor encoder startup alone proves a working control endpoint or
accepted video. The [benchmark runner review](../experiments/vr04-air-encoder-benchmark.md#current-runner-and-result-interpretation)
documents the related counter and verdict limits.

## Verification boundary

Offline checks used exact selected functions and the direct feeder block with
mocked device calls, condition waiting and minimal host state. They reproduced
partial bitrate/VBV changes, feeder-first FPS failure, reference release after
failed `STREAMOFF`, stale running state and zero-reference credit after both
tiles retire, and a startup request left pending after timeout. The harness
passed AddressSanitizer and UndefinedBehaviorSanitizer checks.

These tests do not exercise camera DMA, firmware reopening, actual thread races,
socket framing or hardware recovery. The matched kernel's
[codec recovery study](../experiments/vr04-media-validation.md#later-recovery-and-live-control-implementation)
and the [air timing study](../experiments/vr04-air-encoder-benchmark.md#what-the-air-latency-instrumentation-measures)
retain their separate evidence and limits.
