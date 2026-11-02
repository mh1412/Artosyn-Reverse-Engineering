# VR04 receive queues, frame pairing and display retirement

[Systems](README.md) · [Open media overview](vr04-open-media.md) ·
[Playback](vr04-playback.md)

## Scope

This is a source review of MissingLynk revision
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`, covering the complete receiver,
display and shared media-helper implementations. Selected declarations and the
idle-display command dispatch were also checked. It distinguishes implemented
behavior from comments describing earlier versions. The findings do not add
hardware qualification to the [recorded media tests](../experiments/vr04-media-validation.md).

## Datagram admission and decoder startup

The receiver binds UDP 10001 on all local addresses and treats each datagram as
one access unit. It checks the 36-byte header's magic, header marker, CRC,
channel index and declared payload length. It does not inspect the separate
marker after the HEVC payload, authenticate the sender or require an exact
datagram length. The header's IDR flag, timestamp and resolution are ignored;
PTS/DTS are synthesized from FrameId at a fixed 60 fps. The
[wire-format study](../protocols/vr04-video-session.md#video-record-layout-and-transport-size)
describes the sender's fuller record.

The length guard adds 36 to an unsigned 32-bit length before converting the
sum to `ssize_t`. That addition can wrap. For example, the extracted expression
rejects length 1 in a 36-byte datagram but admits `0xffffffe0`, whose sum wraps
to 4. This verifies a defect in the guard, not successful decoding or a live
network attack. Downstream use of the declared length cannot rely on that check
as a complete payload bound.

The startup helper named `au_has_idr` accepts any encountered VPS (NAL 32) or
IDR (19/20). A VPS alone can open the gate without an IDR picture. Queue-loss
recovery uses a different helper: it skips non-VCL units and accepts the first
VCL only when its type is 16–21, including CRA. These are different admission
rules and neither is a complete HEVC validator.

Prefix-SEI telemetry scans for literal `BR` and `QP` tokens before the first
VCL unit, interpreting bitrate as decimal and QP as hexadecimal. It does not
validate the SEI payload type, UUID or declared payload size, remove emulation
prevention bytes, or check numeric accumulation for overflow. A failed parse
leaves the previous values in place. The emitted frame telemetry adds the two
cached bitrate values and reports tile 0's QP; it is not a fresh synchronized
measurement of both encoders on every frame.

## Input pressure and session continuity

The 4 MiB appsrc policy is a threshold checked before enqueueing, not a strict
memory cap. The check does not add the incoming access-unit size. On reaching
the threshold it drops that unit, even if it is an IRAP, and enters a
drop-until-IRAP state. The next accepted IRAP clears that state and bypasses
the queue-level check for that call. Consequently, a below-threshold push can
overshoot, and resynchronization can enqueue into a still-full queue.

Two additional holds sit outside appsrc:

| Hold | Admission and release |
| --- | --- |
| Tile 1 startup, four access units | Waits for usable tile-0 output; flushes when that output is observed on a later tile-1 datagram, or when the hold fills |
| Tile 0 flow control, 256 access units | Holds while output PTS skew or appsrc buffer count exceeds the configured gate; drains at most eight units per eligible received datagram |

When the tile-0 hold fills, the receiver pushes the entire hold through the
per-unit queue policy and then processes the new unit. That deliberately bypasses
the skew gate, so the configured skew is not an unconditional bound. Held units
also do not drain merely because the decoders catch up during a receive timeout;
the drain is driven by incoming datagrams. Queued bytes, held units, decoded
samples, pairing slots and display references are distinct memory populations.

A received FrameId more than eight below the last accepted global FrameId
advances a timestamp epoch and closes both startup gates. Smaller regressions
do not trigger that reset. The epoch policy avoids the usual new-session wrap;
it does not enforce monotonic PTS for every reordered datagram. Large
cross-channel skew or reordering can also resemble a session change.

The epoch transition does not flush the decoders, clear pairing slots or holds,
or re-arm the initial tile-1 stagger. Old work can drain while the new epoch
starts. A retained NULL/PLAYING rebuild helper is unused in this revision;
it is not an automatically selected parameter-change recovery path. Its older
comments about swallowing packets during rebuild do not describe the active
receive loop.

## Pairing and tile layout

Usable output is paired by exact PTS in a 32-slot table. A missing half keeps
the slot incomplete. When slots or composite buffers run out, the receiver can
evict the oldest occupied slot and retry allocation; continued pool starvation
drops the new half. This bounds retained slots but does not guarantee every
picture is displayed.

In direct-plane mode, each slot retains decoder samples until the pair transfers
to display ownership. Duplicate halves replace the earlier sample. Tile geometry
is taken from the first successful sample per channel, and framebuffer imports
are cached by DMA-buffer file descriptor. This assumes a stable recycling pool;
there is no per-frame geometry or buffer-identity revalidation of a cache hit.

In composite mode, the callback copies each tile into a pool buffer and releases
its decoder sample after the copy path. It tries direct DMA, then staged DMA,
then CPU copying. Seam blending is conditional on geometry and successful band
capture. Staged or CPU tile-1 fallback uses overwrite placement and makes that
pair ineligible for blending. Sharing a FrameId does not make independently
encoded overlap pixels identical.
The [compositor review](vr04-compositor.md#failure-after-blending-does-not-discard-the-picture)
also distinguishes a pre-blend skip from a failed post-blend cache clean: the
latter leaves CPU writes in place, and the caller still submits the picture.

The layout helper prefers populated `GstVideoMeta`, then a usable caps-derived
video layout. Its last fallback uses width as luma stride and places chroma at
`width × visible height`. For the recorded 1920×552 tile, that is byte
**1,059,840**, not the offset after 560 rows. A source comment attributes this
choice to clean retained renders and color bands with the aligned-height
interpretation. Allocation padding is separate from plane placement; this does
not replace the [encoder input-layout rules](../experiments/vr04-media-validation.md#reported-decode-and-encode-measurements)
with a universal visible-height rule.

DMA import requires exactly one DMA-buffer memory object. Its base offset is
added to the plane offsets. Multiple-memory and ordinary mapped buffers leave
the DMA descriptor unavailable and use the appropriate copy fallback. A positive
luma stride selects the metadata branch; the helper does not independently
validate every plane extent against the mapping size.

Completed composites are submitted to display before encoder handoff. An extra
reference protects the encoder call after display takes ownership. Optional
OSD and latency-counter writes precede that handoff, so recording and display
consume the same composite, although either consumer can have its own drops.

## Display mailbox and retirement

The display retains four roles:

| Role | Ownership |
| --- | --- |
| `next` | Newest completed frame waiting to submit; a newer frame releases and replaces it |
| `pending` | Submitted flip awaiting an accepted event |
| `front` | Most recently promoted frame |
| `prev` | Extra retained generation protecting against event/latch uncertainty |

After the first blocking modeset, submissions can run immediately from the
producer or at the end of a flip callback. A display mutex protects the mailbox
transition and submission gate. A failed flip submission releases that candidate;
it is not automatically retried. The newest-wins mailbox can discard a fully
paired frame before scanout, so a composed-frame counter is not a displayed-frame
counter.

An accepted flip event releases `prev`, moves `front` to `prev`, moves `pending`
to `front`, and clears `pending`. This intentionally holds one extra generation
because the reported software event can precede the physical address latch.
Timing marks use callback-time monotonic time, not the DRM event's supplied
timestamp; callback scheduling is therefore part of those marks.

## Event identity and the ineffective timeout fallback

The DRM broker keeps its file description open across process lifetimes. The
display drains already queued events and uses the process PID as event user
data, avoiding dereferencing a pointer left by a dead process. The callback
rejects data that does not equal that token. The token does not identify an
individual flip, and it stays the same across display reinitializations within
one process. A late event from an earlier graph in the same process is therefore
not distinguished by this check. The callback also rotates the held frames when
the token matches but `pending` is empty.

The polling loop checks a pending flip against a nominal 100 ms timeout, with
up to 100 ms polling intervals. Its intended force-retirement call passes the
**context pointer**, while the callback expects the **PID token**. With those
values unequal, the callback returns before clearing `pending`. Subsequent
submissions remain gated. Thus the timeout comment's promised recovery is not
implemented by this call; it cannot be cited as a 100 ms recovery guarantee.
A later real event or another state transition is a separate possible exit.

## Idle display and shutdown limits

The idle-display command raises a flag outside playback; it does not itself
prove signal loss or a completed display transition. The display attempts a
blocking splash modeset/atomic commit and waits two nominal frame periods before
releasing its held frames. If no splash exists, it leaves the previous frame;
the caller nevertheless marks idle as shown. Commit/allocation failures also
do not prevent that software flag, and failed commits do not stop the subsequent
buffer release.

Shutdown stops and joins the display thread, then attempts to park the primary
or disable the tile planes before removing framebuffer imports and references.
It waits four nominal frame periods for primary parking or two for plane disable,
without checking those calls' results or obtaining a hardware-retirement fence.
These waits describe the chosen lifetime policy, not proof of safe quiescence
after a failed ioctl or stalled display. Framebuffer caches are cleared for
normal swaps; partial initialization failures do not have a complete unwind
path in the inspected initializer.

## Broker lifetime, splash adoption and orphan cleanup

The DRM broker retains the shared file description across pipeline processes.
Framebuffer and GEM references can therefore survive a pipeline crash. The
cleanup implementation attempts to recover those resources before allocating
a new composite pool; it is separate from normal frame retirement.

The persistent splash ID is published with the broker socket's inode and
`ctime` seconds/nanoseconds. That is inode change time, despite the source
comment calling it creation time. Adoption requires a matching token and a
1920×1080 YUV420 framebuffer. It does not verify the pixels, plane layout or an
asset hash, so this is a reuse check rather than complete splash identity.
The small state file is overwritten directly with unchecked write/close results.
`ML_NOFBADOPT`, by presence, disables adoption for comparison runs.

If adoption fails, a new splash buffer is allocated. The image loader accepts
the first full image-sized prefix, falling back to black/neutral chroma on a
short read. A failed mapping does not stop the subsequent framebuffer creation;
nor is the flush result checked. Successful allocation/import alone is not proof
that the expected splash pixels reached scanout.

Orphan discovery selects imported YUV420 framebuffers not tracked as the current
instance's splash, primary, pool or tile imports. It uses `MAP_DUMB` failing with
`EINVAL` plus successful export as an imported-object heuristic. This is specific
to the assumed driver behavior. It excludes NV12 imports and native XRGB/ARGB
objects, and does not independently prove that every candidate belongs to a dead
process. Another live producer sharing the broker is outside that ownership
assumption.

The scan is bounded to 64 candidate framebuffers. Handle probing tests IDs
1–4095 and retains at most 256 imported handles, matching them by exported
DMA-buffer inode. Candidate discovery uses the first framebuffer-plane handle;
it is not a complete inventory of arbitrary multiplane backing objects. Binding
classification follows the selected CRTC and stores one plane ID per candidate,
not all possible simultaneous bindings.

Unbound candidates are removed first. For an overlay candidate, cleanup attempts
plane disable, waits two nominal frame periods and removes it without checking
disable success. For a primary candidate, it needs an idle framebuffer and
checks the parking modeset; failure leaves that candidate untouched. Successful
parking is followed by a four-period wait. These fixed waits do not establish
hardware quiescence when display progress is stalled.

Framebuffer removal and handle-close return values are ignored. The returned
“reclaimed” count and logged handle count measure attempts. Multiple framebuffers
sharing one inode can cause repeated close attempts and duplicate byte accounting;
their summed sizes are not unique memory released. The accompanying `CmaFree`
readings are whole-system observations and do not measure MMZ reclamation.

An offline probe returned three reclaimed framebuffers after every mocked
removal, handle close and overlay disable failed. It also counted a shared handle
twice. A separate failed primary-parking case correctly retained its candidate.
These checks verify the counters and branch differences without touching DRM
objects or testing actual resource recovery.

## Offline verification

Exact function bodies and the length-guard expression were extracted unchanged
into a host harness with minimal types and mocked surrounding services. C11
warnings-as-errors and AddressSanitizer/UndefinedBehaviorSanitizer checks passed:

- The timeout's context-pointer call left all display roles unchanged; a
  matching token rotated them, including a second call with empty `pending`.
- VPS-only startup and CRA-only IRAP admission followed the different rules above.
- A push below 4 MiB overshot the threshold; the resynchronizing IRAP bypassed
  the still-full queue check.
- The wrapped declared length passed the isolated size expression.
- BR/QP extraction accepted bytes without a structured SEI payload or UUID.

The harness did not build the complete application, send network traffic,
exercise concurrency, validate the target ABI or run DRM/decoder hardware. It
tests the stated local branches; it does not establish their frequency in
ordinary recorded sessions.
