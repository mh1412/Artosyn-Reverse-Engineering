# VR04 reference codec, memory and DMA validation

[Experiments](README.md) · [Reference board](../devices/betafpv-vr04-reference.md)

Related studies cover the [stock MPP graph and ABI](../systems/vr04-stock-media.md),
[camera-free air encoder benchmark](vr04-air-encoder-benchmark.md) and
[receiver-local display latency](vr04-receiver-latency.md).

## Evidence and chronology

This consolidates the retained MissingLynk kernel capability, encoder-reuse and
patch records at `e3885c6394a7025eacd3a23f473276d9823beb25`. It distinguishes
reported hardware measurements, older failure investigations and the later
implementation present in that snapshot. No hardware tests were rerun here.

The goggle and NT99235 reference air unit have different codec builds and memory
pools. Their results do not identify the Ascent decoder firmware or establish
the SC231AI P1's current behavior. See the separate
[P1 source-stack validation](p1-source-stack.md) and
[Ascent slice experiments](ar9311-slice-decoding.md).

## Capability words and firmware identity

The saved vendor query and open-driver report identify WAVE521C:

| Query register offset | Saved value | Interpretation |
| --- | --- | --- |
| `0x11c` | `0x57415645` | Product name WAVE |
| `0x120` | `0x0000521c` | Product WAVE521C |
| `0x118` | `0x00050b8e` | Vendor-runtime firmware revision 330638 |
| `0x140` | 0 | Customer field |
| `0x124` | `0x000d0980` | Single combined backbone; separate VCPU/per-core backbone bits clear |
| `0x128` | `0x08818003` | Encoder bits 0/1 set; decoder bits 2/3 clear in the interpreted W521 layout |
| `0x12c` | `0x00000507` | HEVC/AVC 10-bit encoder flags clear |
| `0x130` | `0x01343ca7` | Decimal 20200615 configuration date |
| `0x134` | `0x00031345` | RTL configuration revision 201541 |

The extracted open-driver blob reports revision **329715**, not 330638. Correct
decode on both does not make that difference universally cosmetic. The driver
enables decode despite the capability-word bits because actual decoding was
demonstrated. That is a qualified integration quirk, not permission to ignore
capability words on other products.

The validated path is 8-bit. A Main10 menu entry does not establish 10-bit output,
and no Main10 decode result was supplied. Likewise, advertised sizes up to
8192×4320 are driver ranges, not measurements above 1080p. Offered alternate
pixel formats are distinct from the validated I420 path.

## Reported decode and encode measurements

| Operation | Reference result |
| --- | --- |
| HEVC decode 640×360 and 1280×720 | Eight of eight frames each; zero mean absolute difference in Y/U/V against host decode |
| HEVC decode 1920×1080; H.264 decode 1280×720 | Reported byte-exact output |
| Decode to discard sink | 323 fps at 720p; 165 fps at 1080p |
| HEVC encode 720p / 1080p | Reported PSNR ranges 42.6–45.3 / 43.7–48.1 dB |
| H.264 encode 720p / 1080p | Reported PSNR ranges 42.4–44.7 / 43.3–46.0 dB |
| Encode to discard sink | 171 fps at 720p; 108 fps at 1080p, including test-client source copies |

Discard-sink throughput is not optical latency, a complete camera/display rate,
or a guarantee for arbitrary streams. The encode report is internally inconsistent:
its matrix introduction says rate control enabled and labels 5/8 Mb/s, but its
later control section explicitly says the measurements used **fixed QP 30**,
yielding roughly 1.5 Mb/s at 720p and 4–5 Mb/s at 1080p. Retain the quality
observations as reported; do not label them verified CBR results. Its additional
1080p HEVC “all planes 47+” phrase also conflicts with its stated 43.7–48.1 range.

In the recorded API, `FRAME_RC_ENABLE` defaults off. Setting BITRATE alone then
has no targeting effect. Framerate uses OUTPUT `VIDIOC_S_PARM`. Source-plane
offsets follow negotiated stride and aligned height, not visible geometry:
1080 rows can require 1088 source rows. Crop/conformance and coded height must
remain distinct from allocation geometry. Later patches also preserve a wider
2048-byte CVISP stride and imported planes' `data_offset`.

## Pool capacity and buffer ownership

The reference goggle reserves 110 MiB for media, compared with a 32 MiB air
pool at `0x25000000`. One concurrent test used two HEVC 960×1080 decoder instances
and a 1920×1080 H.264 encoder:

| Consumer | Reported working allocation |
| --- | --- |
| Each decoder, including nine linear CAPTURE buffers and FBC references | Approximately 36.5 MiB |
| Encoder, eight FBC buffers and source queue | Approximately 16 MiB |
| Combined working set | Approximately 92 MiB, plus about 3 MiB common/firmware |

The source calls the available pool about 108 MiB in this accounting; that is
not a replacement for the DT's 110 MiB reserved extent. The eight-second run
encoded **480/480** composites with zero reported drops and a host-decodable MP4.
This geometry is one test setup, not the top/bottom tile layout of every RF mode.

Power-of-two coherent allocation previously made a 3,133,440-byte FBC allocation
cost 4 MiB and a 5,041,296-byte task allocation cost 8 MiB. Page-granular first-fit
and explicit per-allocation page counts allowed the concurrent load to fit.
The patch changes **all per-device coherent pools** and deliberately loses their
natural power-of-two alignment guarantee. Clients that mask address bits for
descriptor rings cannot assume that alignment under this allocator.

Reducing the linear decoder queue below the stream's FBC-buffer count is not
equivalent to reducing latency. With a count of nine in the tested 1080p60 stream,
an undersized queue starved firmware display rotation at roughly 25–40 fps and
pegged both CPUs. The report keeps diagnostic `dec_cap_bufs=0`. Deep-reference
B-picture streams can still exceed the pool; the successful workload used
low-latency streams.

## Encoder reuse: retained failures

The original investigation found clean, watchdog and corrupt-output outcomes
after destroying and recreating encoder instances within one firmware boot.
The watchdog result was `0x20000`, around 125 ms into the first PIC_RUN. Another
failure completed but emitted deterministic noise, approximately 200 times the
healthy size. Successful dequeue, frame count and file size were insufficient.

| Goggle experiment | Recorded outcome |
| --- | --- |
| HEVC 1080p, 60/600/1500 frames per instance | Alternated good/bad |
| Same, six seconds idle between instances | Still alternated |
| HEVC 720p | Alternated |
| HEVC 640×480 | Six of six clean, with/without a live decoder |
| H.264 720p GOP60 | Six of six clean and byte-identical, decoded correctly |
| H.264 1080p GOP60 | Clean continuation after the first instance |

On the air unit, some frame counts chained clean while others poisoned the next
instance, and H.264 could perpetuate corruption. That differs from the goggle's
clean H.264 chain. Different driver builds are a candidate explanation, not a
proven cause. Under 10 Mb/s rate control, a poisoned goggle HEVC instance even
reported 60 coded frames and emitted 1,411,202 bytes of bad output.

The research checked identical allocation addresses/sizes, visible queue/status
registers, forced runtime power-on, explicit STREAMOFF versus close, a surviving
sibling instance and synthetic camera-free input. None removed the pattern.
It attributed the behavior to private running codec state; the exact state and
one model explaining both boards remained unresolved. Decoder churn did not show
the same failure in the recorded tests.

The DVR record is weaker than a controlled same-boot reproduction. H.264 files
decoded clean, and several HEVC files were large/nonempty, but boot attribution
was missing; other files were empty or header-only. Three attempted repeat-HEVC
tests had no RF input and `pushed=0`, so they did not test encoder reuse. The later
policy defaults DVR/RTSP to H.264 with HEVC opt-in. A file existing is not proof of
correct encoding or a resolved instance-lifetime defect.

## Later recovery and live-control implementation

The early reuse report described warm reload failing with `-16` and an in-driver
reset/reload as unbuilt. The retained patch series **does implement** that path:
it removes the stale-PC initialization rejection and uses `SW_RESET_ON_BOOT`
plus firmware download. Sleep/wake-based safety reset had timed out before the
reset itself. Recovery is global and respects live-instance constraints.

If close cannot quiesce DMA, it marks `keep_dma_bufs` and retains the affected
allocations. This is deliberate containment, not leak-free recovery: freeing
memory still owned by the codec had hung the SoC. File-release locks also became
uninterruptible to avoid SIGKILL skipping cleanup. These changes supersede
“recovery is unimplemented,” but are not a new repeat-encode qualification of
every historical corruption case.

Two independent wait boundaries were added: a 500 ms unanswered-job timeout and
a 2000 ms STREAMOFF drain budget. The job timeout uses the IRQ lock, not the
device lock held by the waiter. A recorded 124-second drain across two passes
showed why fixing only the active-job wait was insufficient. Lost-completion
injection and debugfs state/recovery paths were added for diagnosis.

Bitrate, VBV, frame rate and forced IDR changes use `W5_ENC_SET_PARAM` between
pictures. Its completion report must be drained with GET_RESULT before later
picture results, or the first change stalls the queue. The working IDR route
sets intra period to one for one picture, then restores it; the tested picture-
parameter force bits had not produced an IDR. The patch record reports five
bitrate steps and recovery of a receiver joining mid-stream.

The old capability note's blanket “parity parameters not written” also predates
the patch content. The inspected encoder patch sets maximum HVS QP delta to 4,
RDO skip and lambda scaling to 0, buffer weight to 1, temporal-MV prediction to 1
and merge count to 2. It retains `hvs_qp_scale=2`. An old request for scale 4
must not be presented as the implemented value or a universal hardware constant.

## Related DMA and USB failures

The same patch record describes three independent AXI DMA defects:

1. **Lost controller interrupt enable.** Concurrent register read/modify/write
   could lose `INT_EN` while transfers continued. Two incidents followed about
   200,000 compositor interrupts, with `DMAC_CFG=1`, `DMAC_INTSTATUS=3`, frozen
   IRQ counts and no log error. Restoring the bit returned frame time from
   4091.6 to 16.6 ms. A controller-wide lock replaced reliance on per-channel locks.
2. **Error reported as success.** A refused descriptor previously completed as
   though copied, exposing stale pixels. The patch reports `DMA_TRANS_ABORTED`;
   the client receives `-EIO` and can copy that plane on the CPU. Three errors
   were recorded during one hour-long seam-band workload.
3. **Rapid channel re-enable.** Three separate plane copies could re-enable the
   channel within microseconds during display fetch and hang the memory fabric.
   The report measured 2.48 hangs/hour. Batched memcpy builds one linked-descriptor
   chain with one enable/completion, matching the vendor compositor's scheduling
   shape. This is separate from the lost-interrupt race.

USB had another failure: sustained gadget bulk OUT alongside decode wedged the
DWC2 descriptor-DMA ring, timed out endpoint stopping, re-enumerated USB and
starved video during recovery. The patch selects **buffer DMA**, keeping DMA
enabled, instead of descriptor DMA. It does not turn the gadget into PIO or prove
that every USB disconnect has the same cause.

These records explain the implementation and observed failure signatures. They
do not replace workload-specific qualification of a newer board, firmware or
complete pipeline.
