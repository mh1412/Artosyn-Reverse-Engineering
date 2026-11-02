# G11 stock video-control socket

[Protocol index](README.md) · [Vendor SDK scope](../systems/g11-vendor-sdk.md)

Scope: VideoPlayer source in the August 26, 2026 vendor SDK checkpoint
`b3917ed3c3a0eef0afc6548ba6a00cffc61d2293`. This is local process control on the
Linux host, separate from Artosyn host/RF framing and from the compressed-video
UDP stream. Source behavior is not assumed identical to every installed player.

## Transport and header

The server binds Unix-domain **stream** socket `/tmp/video_control.sock` and
keeps one active client. The documented HAL pattern uses short connections;
the server itself can process multiple messages on a retained connection.
It serializes native C structures without byte-order conversion or a checksum.
The ARM build uses four-byte integers/enums and little-endian words.

| Header offset | Bytes | Field |
| --- | ---: | --- |
| 0 | 4 | Magic `0x56494445` |
| 4 | 4 | Caller sequence number, echoed in reply |
| 8 | 4 | Command enum |
| 12 | 4 | Payload byte count, maximum 512 |

The header is packed to **16 bytes**; maximum message size is **528 bytes**.
Although the constant is labeled “VIDE,” the serialized little-endian bytes
are `45 44 49 56`. For example, GET_STATUS with sequence `0x12345678` is:

```text
45 44 49 56  78 56 34 12  04 00 00 00  00 00 00 00
```

## Commands and payloads

| Command | Request body | Reply body under command 5 |
| --- | --- | --- |
| 1 SET_PLAYMODE | 264 bytes: mode, two URL arrays, UDP port | 68-byte ordinary ACK |
| 2 SET_DISP_STATUS | Four-byte integer | Ordinary ACK |
| 3 REQUEST_RTSP_SWITCH | Four-byte signed index; a negative value asks the server to toggle | Ordinary ACK |
| 4 GET_STATUS | Empty | **272-byte status**, not an ordinary ACK |
| 5 ACK | Reply command | Body layout depends on the request |
| 6 HEARTBEAT | Empty | Ordinary ACK with `pong` text |

| SET_PLAYMODE offset | Bytes | Field |
| --- | ---: | --- |
| 0 | 4 | Mode: 0 none, 1 RTSP, 2 UDP, 3 picture |
| 4 | 128 | Primary URL |
| 132 | 128 | Secondary URL |
| 260 | 4 | UDP port field |

The HAL's public RTSP argument has two **64-byte** arrays; these must not be
confused with the socket payload's two 128-byte arrays. The source server keeps
larger 256-byte URL slots internally. Use terminated strings within the
actual boundary's capacity.

An ordinary ACK is signed four-byte result code plus a 64-byte message.
Status contains four words—mode, display state, stream-active state and current
RTSP index—followed by two 128-byte URL arrays. A client must interpret command 5
using the outstanding request, sequence and exact payload size. Treating every
ACK as a result-code/message structure would misread a status reply.

## What the source actually applies

SET_PLAYMODE updates mode state, pauses the previous network source and
switches the display toward the picture path before resuming UDP or RTSP.
Selecting the same non-RTSP mode returns immediately. RTSP mode can restart
with a supplied URL; empty URL fields leave the corresponding stored URL intact.
Picture mode toggles the decoder's output-buffer control. None of these ACKs
establishes received frames, successful decoding or visible pixels.

The handler **does not use the request's UDP port**. The inspected UDP worker
binds port **554** directly. A request containing another port can therefore
be acknowledged without rebinding the listener. The custom UI deliberately
retains that [fixed-port behavior](../systems/g11-linux-ui.md#video-controls-and-alpha).

RTSP switch is rejected while in UDP mode. A negative index toggles between
the two stored URLs; an explicit index must be 0 or 1 with a nonempty URL.
The switch helper also writes a legacy flag, but the active legacy reader
only recognizes reboot. Its old RTSP-switch reader is commented out, so the
file interface is not an equivalent implementation of all socket commands.

Display-off calls the screen-off routine. The corresponding screen-on call
is commented out: nonzero status updates the software field and still receives
an ACK. This is another reason to distinguish reported state from physical
display behavior. It is also a different interface from PWM brightness control.

## Stream activity is not decoded-picture evidence

The UDP worker calls `update_video_heartbeat()` immediately after a positive
receive, before NAL parsing or decoder submission. Even malformed or incomplete
video input can therefore refresh activity. The reviewed RTSP decode worker
calls it after receiving a decoded frame, illustrating that activity's
meaning depends on the source path.

The [complete ingestion-worker review](../systems/g11-stock-video-ingest.md)
adds UDP parser truncation, parameter-cache failure handling and RTSP queue,
pause/rebind and synchronous URL-switch limits. Its heartbeat occurs before
software display submission and does not establish physical scan-out.

The monitor nominally polls every 200 ms and compares integer wall-clock seconds
using **elapsed > 3**, not a precise monotonic three-second deadline. It also
sets activity true when the timestamp is recent, even without a new heartbeat.
A source-fragment replay initialized UDP mode with no packets and reproduced
that startup activity transition. Its printed “Video data resumed” text did
not establish traffic.

After timeout, a resume counter waits for three heartbeat calls. There is no
inter-call freshness check in that counter: a separate replay with calls
50 seconds apart still resumed on the third call. For UDP these are receive
events, not necessarily three complete frames. Those synthetic replays replace
clock, IPC polling and hardware-command effects; they are not device tests.

The [physical video experiments](../experiments/g11-video-and-debugger.md)
instead compare decoder state, dimensions and captured decoded pixels. They
also retain an installed-player observation where activity stayed stale after
selecting mode 0. The archived source and that particular installed behavior
must not be collapsed into one fully verified implementation.

## Blocking and validation limits

The listening socket is nonblocking, but accepted client sockets are blocking.
Receive first uses `select`, then `MSG_WAITALL` for the full header and payload.
Its timeout bounds initial readability, not completion of the entire message.
An offline socket-pair test of the actual IPC source sent one header byte:
`video_ipc_recv(..., 10)` remained blocked after 100 ms and completed only when
the other fifteen bytes arrived. A slow partial message can therefore stall
the monitor despite the “nonblocking” comments and zero-timeout poll call.

Send performs one `send` and reports failure on a short write, without completing
the remainder. Receive rejects wrong magic and payloads above 512 bytes, but
the command handler does not enforce each command's required body size, valid
mode range or URL termination before casting/accessing the body. These are
source-level input-validation limits, not tested hardware fault recovery.

Connection retry performs five attempts with sleeps of 0.5, 1, 2, 4 and 8 seconds,
including the final failure sleep: **15.5 seconds of sleep** plus connection
attempt time. It is not a fixed five-times-500-ms timeout. The later custom UI
uses its own bounded worker and reply checks; that does not change server code.

## Offline verification

The exact header/payload sizes and offsets compiled with both the host compiler
and the SDK's ARM GCC 8.2 toolchain. A host socket-pair probe of the unchanged IPC
source verified framing, empty-input timeout, close handling, bad-magic/oversize
rejection and the partial-header blocking case. The two monitor findings used
exact extracted function bodies with side effects stubbed. No socket on a G11,
radio, decoder or physical display was accessed.

Reviewed IPC source SHA-256:
`ccca17806c32f115b838842e38a32d9402b32bc37e42d619511575ed6dbf141b`.
Monitor source SHA-256:
`9a95203635312c05ba3adf0429ce941e91a24e515cb3bfffb7edd6497e199b81`.
