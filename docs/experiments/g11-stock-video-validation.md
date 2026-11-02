# G11 stock player decoded-pixel validation

[Experiments](README.md) · [G11](../devices/skydroid-g11.md)

On October 26, two captures from the installed vendor player's VDEC output
matched source frames byte for byte at 1280×720 NV12. This validates that player
and test stream, not an end-to-end stock handset configuration or physical LCD
composition. The original fallback picture was not verified restored.

## Bench transport and observations

The installed stock player accepts Annex-B HEVC over UDP 554. Its supplied SDK
initializes HEVC; H.264 is not interchangeable. The bench sender groups complete
access units by AUD, accepts 100–29,000 bytes per unit and sends one unit per
UDP datagram. It appends a trailing zero to compensate for the SDK parser
discarding the last byte at its buffer boundary. There is no RTP or MPEG-TS
container. Large datagrams can fragment, so this is a finite bench source,
not a qualified production transport.

The repeated source was a four-second, 100-frame `testsrc2` sequence encoded
at 1280×720, 25 fps, HEVC ultrafast, 600 kbit/s target/max/buffer, key interval
25, no B frames/open GOP, with repeated headers and AUD. Its SHA-256 was
`c19ec17b05dfbb3ce74d91b460c347c75f4bf18c49866bcc7208a6b699c965b2`.
Later finite runs reused those access units rather than proving arbitrary
stream compatibility.

The video-control socket exposes modes 0 none, 1 existing RTSP, 2 UDP and
3 picture. A mode ACK is not decoding evidence. `stream_active` is a software
activity flag: UDP traffic updates it before parsing/decoding, and the archived
SDK monitor can also set it during a recent-timestamp startup window without
new traffic. The [source review](../protocols/g11-stock-video-control.md#stream-activity-is-not-decoded-picture-evidence)
distinguishes those paths. On the recorded installed player, the flag could
remain stale if mode 0 was selected before the no-data timeout.
One recorded recovery waited 4.5 seconds in UDP mode without a sender,
confirmed inactive, then selected mode 0.

Decoder dimensions and decoded/displayed counters are useful together. The
idle player can repeatedly decode a fallback image, so nonzero counters alone
do not prove network video. The 720p source distinguished itself from that
fallback. Graphics framebuffer captures contain only the UI plane, not the
decoded picture beneath it.

## Capturing actual decoded pixels

The successful helper used the vendor MI_SYS output-port API rather than the
display screenshot wrapper. It required VDEC0 bound to DISP0, user depth 0,
queue quota 3 and no user-held frames. It temporarily selected user depth 1,
waited at most three seconds, validated linear NV12 metadata, copied planes,
released the frame and restored depth 0/quota 3. It did not initialize HAL/SYS,
open the MCU UART, restart decoding or alter display binding. Forced process
termination could prevent cleanup, so successful release/close/restore and
final port state were part of the recorded result.

Three captures passed all those checks. The idle image was an orange landscape
fallback at **1920×1056**; allocation dimensions must not be substituted for
visible dimensions. During a 75-second, 1,875-frame 720p25 run:

| Capture | Matching source frame, zero-based | MSE | Raw NV12 SHA-256 |
| --- | --- | --- | --- |
| A | 3 | 0 | `6b90d747e0714c43b55ca509ebdcb97c849e48fa2453d7071e5198fa544b3c27` |
| B | 54 | 0 | `0059a06acd1fcbd497f1ba643573480b5ac43cf306be7c1e9bf5b54f49d1637a` |

Both were byte-identical among the 100 reference frames, stronger than merely
meeting the comparison's MSE threshold of 4. Each capture was 1280×720 linear
NV12 with stride 1280 for both planes. The reported allocation was 1,413,120
bytes; tightly packed visible pixels use 1,382,400 bytes. Allocation padding
must not enter the visible-image comparison.

## Restoration and capture limits

Mode 0 and `stream_active=0` were restored, but the decoder retained 1280×720.
The original fallback picture was therefore not established restored. The
supplied SDK's picture-switch function sets a flag that pauses its picture
thread without an active reset path, a candidate explanation rather than proof
of installed binary behavior. No player restart was performed for that check.

The [complete source lifecycle review](../systems/g11-stock-display-lifecycle.md)
also finds that the no-stream file-switch block is disabled and the readiness
helper destroys/recreates decoder channel 0. These source findings explain why
a mode change alone is insufficient evidence of fallback restoration; they do
not add a physical restoration test.

The installed display library matched the inspected artifact
`3bae0eaab3df21fcc4b3edd521639d6e79574ce66ad85adfaa6c4f767a1f6a05`.
Its screen-frame wrapper can replace a failed ioctl status with zero without
populating metadata. That explains why return zero was insufficient, but not
the precise driver error behind earlier empty captures. VDEC capture avoided
that wrapper.

The final 1920×1080 graphics framebuffer was clean. Decoder-plane identity
and graphics-plane correctness remain separate from the physical LCD's final
blending/transparency. Long-term load behavior, arbitrary video, physical display
composition and external radio delivery remain open.
