# G11 stock UDP and RTSP video ingestion

[Systems](README.md) · [Video-control protocol](../protocols/g11-stock-video-control.md) · [G11](../devices/skydroid-g11.md)

Scope: the VideoPlayer source retained in vendor SDK revision
`b3917ed3c3a0eef0afc6548ba6a00cffc61d2293`. The complete UDP worker, RTSP
worker, shared NAL parser and application entry point were reviewed. This source
is not established as byte-equivalent to the player used in the
[physical decoded-picture experiment](../experiments/g11-stock-video-validation.md).

## UDP transport and parser boundaries

The worker binds IPv4 UDP port **554** on all local interfaces. It receives
compressed Annex-B bytes directly, without an RTP depacketizer, sequence-number
tracking, sender filtering or an application-level loss/reordering mechanism.
The source endpoint returned by `recvfrom` is not used to select a stream.
Playback-control requests do not change the fixed port.

Received datagrams are appended into a 1 MiB buffer. Parsing is deferred while
the accumulated length is below 100 bytes **or the latest receive exceeds
30,000 bytes**. This is a size heuristic, not an access-unit delimiter or a
time limit. The one-second socket receive timeout does not flush a short
accumulation: its timeout branch sleeps 10 ms and continues receiving.

Once parsing starts, `GetAnnexbNALU_from_buffer` searches for three- or
four-byte start codes. A following start code correctly delimits the preceding
NAL. At the end of the currently available buffer, however, the parser returns
`pos-1` bytes and advances the offset to the end. Unlike the file reader's EOF
case, no failed byte read occurred: **the last actual byte is omitted**. It
also treats that tail as finished rather than retaining it for the next datagram.

An offline replay of the unchanged parser demonstrated:

| Input | Result |
| --- | --- |
| One eight-byte NAL ending at the buffer boundary | Seven bytes returned; offset consumes all eight |
| Two eight-byte NALs | First returned intact; final NAL loses its last byte |
| A bare four-byte start code | Positive return of three bytes despite no NAL header |
| Resynchronization target consisting of a three-byte start code and one header byte at the tail | The worker's resynchronizer misses it |

The worker does not validate the returned NAL's header length before examining
its type. A short successful parser result can therefore make type selection
depend on bytes outside the newly returned content. On parser failure, the
resynchronization loop advances or skips the remainder; it does not preserve
all possible partial start codes across receives.

The parser allocates and frees a 1 MiB scratch area for each call. The worker
also allocates its receive buffer, NAL buffer and another 1 MiB slice buffer
that is unused by its active path. A near-full receive-buffer reset occurs
after parsing, not as an independent receive deadline.

## NAL dispatch, parameters and GOP cache

Codec selection comes from the decoder configuration, not automatic inspection
of the incoming stream. The worker has these dispatch rules:

| Codec branch | Parameter sets cached | IDR handling | Other slice handling |
| --- | --- | --- | --- |
| H.265 | Types 32/33/34 | Types 19/20 | Types 0–9 |
| Other branch, interpreted as H.264 | Types 7/8 | Type 5 | Type 1 |

Everything else is submitted through the generic NAL branch. In particular,
H.265 type 21 does not trigger the IDR/cache-flush branch. These are source
dispatch rules, not a complete random-access-picture classifier.

Up to ten parameter NALs are stored in arrival order. The cache does not replace
entries by parameter-set ID or remove duplicates; when full, it drops the oldest.
On an IDR it submits all cached parameters with end-of-frame false, frees them,
then submits the IDR with end-of-frame true. Ordinary recognized slices are
also submitted individually with end-of-frame true. There is no picture-header
parsing to establish the last slice of a multi-slice picture.

All submissions use channel 0, PTS 0, end-of-stream false and timeout argument
-1. Callers ignore `send_to_decoder` failure. The replay forced both parameter
submissions and the IDR submission to fail: all were still attempted and the
parameter cache was consumed. The next IDR had no cached parameters to replay.
A special nine-byte pattern beginning `00 00 00 01 68` and ending in zero is
shortened to eight bytes before submission; this rewrite is not codec-gated.

The separate GOP cache deep-copies parameters and slices, avoiding shared
ownership with the parameter cache. It stores at most **500 NAL records**, not
500 pictures or a fixed byte budget. Its oldest-entry eviction can remove the
IDR and parameter records while leaving later slices. A fixture with one IDR
followed by 500 slices reproduced loss of the keyframe record. File-saving
calls are commented out, but allocation and copying remain active. This cache
is not a demonstrated recording or loss-recovery mechanism.

## UDP pause and teardown

Pause sets a flag; it does not close the socket, flush queued datagrams or clear
the accumulated bytes and parameter/GOP caches. An already entered receive or
decoder call can still complete. Resume signals the waiting worker without
reinitializing its stream state.

Initialization returns 0 without checking `pthread_create`. Worker allocation,
socket or bind failures are not communicated through that return. Several
early exits bypass cleanup of earlier allocations. Deinitialization sets a
global exit flag, resumes the worker and joins it; initialization does not reset
that exit flag for a second lifecycle. The receive timeout does not bound a
decoder call or the complete join, and the worker does not recheck exit directly
after waking from its pause wait.

## RTSP connection and decoder selection

The RTSP worker requests TCP transport, `stimeout=5000000`,
`max_delay=500000` and `fflags=nobuffer` from its FFmpeg build. Those option
strings do not prove a bounded end-to-end reconnect or display latency. No
interrupt callback or independent join deadline is installed in this file.
Failed initialization and ordinary disconnects use a fixed two-second retry
sleep.

The last encountered video stream is selected. Audio discovery explicitly
leaves the audio index at -1, so the audio decoder code is not enabled by this
scan. H.264 and HEVC select the named `ssh264` and `sshevc` decoders;
other codec IDs use the generic lookup. The source passes display dimensions
through codec `flags/flags2`, reflecting its vendor integration rather than a
portable FFmpeg configuration recipe.

An old decoder can be reused when codec ID, dimensions and pixel format match,
after updating its parameters. Otherwise it is destroyed, followed by a 100 ms
sleep, a decoder reset and a new decoder allocation. Display/configuration
helper results are not all propagated. `STCHECKRESULT`, defined in the shared
header, logs a failure but does not return from the calling function.

## RTSP queue, decode and recovery limits

The producer transfers packet ownership by copying the packet structure into a
linked queue. It stops reading while the combined recorded queue size is at
least 2 MiB, sleeping 10 ms per check. This is a threshold checked before the
next enqueue, not an exact allocation ceiling. Queue-size reads occur outside
the queue mutex. The producer ignores enqueue failure, leaving no explicit
packet-release or retry path for a failed queue-node allocation.

The consumer sends one packet and requests one decoded frame per iteration.
On send `EAGAIN`, it proceeds to receive but does not retry that packet before
unreferencing it. It does not drain all available frames per packet. Other
decode errors exit the consumer without notifying the producer that its worker
has stopped. The producer's active flag remains set and can keep filling the
queue until its size threshold blocks further network reads. That state has
no autonomous consumer restart in the inspected code.

The `first_keyframe_received` field is initialized/reset but never used to gate
packets. Stop uses abort flags and a poison packet, then joins and flushes the
queue. The poison packet exits the consumer before decoder submission; it is
not an end-of-stream drain that retrieves delayed output.

After a successful decoded-frame receive, the worker updates the heartbeat for
both hardware-codec and software-codec paths. Software output is scaled to NV12
and copied into display buffers as contiguous Y and UV planes without consulting
the destination strides. Display-buffer submission is separate from the earlier
heartbeat. Neither event verifies physical scan-out or optical latency.

## RTSP pause, resume and URL changes

There are two separate startup paths. Application initialization creates a
worker while the global pause flag is initially 1. A subsequent same-URL resume
clears that flag and signals it. In contrast, directly calling
`resume_rtsp_client` when no worker exists creates one without clearing the
initial pause flag. The offline lifecycle fixture reproduced this distinction;
it does not imply that every normal application startup is stuck.

A changed URL updates the shared URL cache, sets the exit flag, wakes and
**joins the old worker synchronously**, then creates its replacement. A mode
ACK cannot therefore be treated as a bounded network-switch deadline when
the old worker is blocked in an external call. The URL cache is also passed
directly as the worker's filename rather than copied per connection.

Pause stops and joins the video consumer but retains the decoder context until
resume. The consumer's exit path unbinds hardware decoding from display. On
resume, initialization reopens the input and may reuse that decoder; binding
is done only in the new-decoder branch, and the consumer's own bind call is
commented out. This leaves no explicit rebind in the matching-parameter reuse
path. It is a source-level recovery gap, not a reproduced LCD failure.

## Application and shared-helper boundaries

The entry point initializes display/decoder services, the monitor and all three
playback modules, including an initially paused RTSP worker. Several initializer
returns are ignored. `--vpath` stores a string and sets a flag, but the active
entry point does not start a file-video worker from it. Help, parse errors and
several invalid configurations return process status 0. The audio/video path
arguments are copied into 256-byte arrays without a length bound.

Shutdown calls the display deinitializer before joining the UDP, picture,
monitor and RTSP modules, then deinitializes the decoder. The
[complete display review](g11-stock-display-lifecycle.md) shows that this first
call performs screen-off and GPIO-direction commands while full display teardown
is disabled. That ordering does not establish that all display users have stopped
before screen-off.

The shared file-based Annex-B reader is separate from the UDP buffer parser.
Its byte-reading loop does not enforce the temporary allocation's maximum size;
its failed-`fseek` branch frees the temporary buffer and then falls through to
the normal copy/free code. These static findings are not evidence that the
active UDP path uses that file reader. The shared time helper uses wall-clock
milliseconds, not a monotonic media timestamp.

## Verification

Host fixtures compiled the unchanged shared parser with substitute dependency
types, the unchanged UDP helper section and the exact RTSP resume/stop function
bodies. AddressSanitizer and UndefinedBehaviorSanitizer checks passed for the
parser boundary, cache eviction, forced decoder-error and lifecycle cases above.
Decoder submission and thread creation/join were stubbed; no network worker,
FFmpeg decoder, actual concurrent lifecycle or G11 hardware was exercised.

Reviewed source SHA-256 values:

| Source | SHA-256 |
| --- | --- |
| UDP worker | `e1ad18c6aa94f5bd20c85750cc8226b6c3dbb8b6800c99d6f8790a726f404cf6` |
| RTSP worker | `f96765eca1751ece5e583efaa94c22b2860d055b09406612722025b034fbccc4` |
| Shared parser/helpers | `0bac68691df870d8b399bf4ae97696d27696b071ce05e393c50c761a4d4e7f30` |

These results qualify the archived implementation. They do not supersede the
separately recorded physical picture comparisons or establish equivalence to
an installed binary, general stream compatibility or complete vendor SDK review.
