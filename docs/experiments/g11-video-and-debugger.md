# G11 decoded video, MCU coexistence and debugger interference

[Experiments](README.md) · [G11](../devices/skydroid-g11.md)

## Strongest recorded result

On October 26, two physical VDEC output captures matched host-decoded source
frames byte for byte at 1280×720 NV12. The autonomous MCU continued approximately
500 Hz mixing during video and a brief Linux UI suspension, with RF off. This
establishes decoded pixel content and bounded coexistence, not physical LCD
composition, receiver delivery or worst-case timing. The original fallback
picture was not verified restored after the test.

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
3 picture. A mode ACK is not decoding evidence. `stream_active` reflects input
traffic and may remain stale if mode 0 is selected before the no-data timeout.
One recorded recovery waited 4.5 seconds in UDP mode without a sender,
confirmed inactive, then selected mode 0.

Decoder dimensions and decoded/displayed counters are useful together. The
idle player can repeatedly decode a fallback image, so nonzero counters alone
do not prove network video. The 720p source distinguished itself from that
fallback. Graphics framebuffer captures contain only the UI plane, not the
decoded picture beneath it.

## Earlier Linux-mixer test, October 24

A 35-second stream established 1280×720 decoding. A second 50-second stream
included 12 menu open/close cycles and layout changes. Over 57.28 seconds of
surrounding observations, 5,467 inputs arrived without new decoder rejects
or UART overruns; the overrun count remained 108.

The 40-second SWD sample retained 1,066 coherent observations and discarded
1,743 snapshots whose accepted sequence/timestamp/flags changed while reading.
Accepted-input ages were median 13.385 ms, p99 19.104 ms and maximum 25.220 ms.
These are sampled ages, not a maximum physical stick-to-receiver delay. SRAM
addresses were derived from the exact installed ELF; they are not portable
between builds. Faster 400 kHz SWD improved coherence relative to 100 kHz but
could still perturb bus timing.

A separate 300 ms UI SIGSTOP/SIGCONT test saw the same accepted sample live
at age 95.352 ms and expired at 106.577 ms, consistent with the 100 ms policy.
The MCU stayed in its application with RF off. Maintenance → resident →
RUN-existing restored input without flash writing or Linux reboot. This tests
the historical Linux-dependent control deadline, not receiver failsafe.

The display-frame API returned zero-sized metadata during idle and streaming;
those were invalid captures even when its return code was zero. Physical
picture correctness remained open at this stage.

## Autonomous coexistence, boot 115

The revision-10 MCU and UI were tested with 1,500 access units over 60 seconds.
The decoder changed to 1280×720 and all sampled control/source/handset data
remained fresh with zero Linux mixer cycles. A surrounding 2.38-second UI pause
included 69/70 additional decoded/displayed frames. Four MCU observations
during the pause bounded its rate at 498.31–500.67 Hz; the full 40-second window
measured 499.96–500.06 Hz. Cumulative peaks were 831 µs mixing and 384 µs
lateness, not worst-case bounds.

One decoder drop appeared at startup and stayed constant. Linux UART breaks
rose from 12 to 16 early, then stabilized. Post-test MCU recovery count was 16
without a pre-test baseline; not all 16 can be attributed to video. Recovery
also resets decoder-error diagnostics, so zero current errors did not prove
an error-free link. MCU/Linux boots, UI PID and management generation stayed
unchanged; no model, RF or firmware mutation was part of the test.

## Debugger interference isolated

The follow-up compared live executable reads against adjacent idle intervals
on the same boot-115 installation:

| Operation | MCU recoveries / Linux breaks added |
| --- | --- |
| Full executable read 1, about 2.742 s | 4 / 4 |
| Full executable read 2, about 2.745 s | 5 / 5 |
| Adjacent idle windows | 0 / 0 |
| Eight-word reads without pauses | 7 / 7 |
| Eight-word reads with 1 ms pauses | 6 / 6 |
| Exact comparison while verified RF-off UI UART owner paused | 0 / 0 |
| Separate 30 s video run with sparse RAM diagnostics | 0 / 0 |

Smaller paced reads did not remove the interference. The updated timing tool
could explicitly pause/resume the verified RF-off UI for identity comparison,
then measure with it running. Its recorded check measured 499.84–500.17 Hz
with no new recovery/break and unchanged boots. Failure-path fixtures checked
cleanup and process-identity changes; other inspection tools were unchanged.

This isolates a repeatable debugger interaction, not the exact UART hardware
error or ring overflow, nor every historical loss. Read-only access is not
necessarily non-perturbing measurement. Earlier connector assumptions in this
report were later corrected by the [RX/PPM scope investigation](g11-crsf-electrical.md).

## Capturing actual decoded pixels, boot 122

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

The installed MCU artifact was
`36af66b3dbcf077438d611ebe2ca4acfdedd8295061ece65cbda34dc7103dae1`,
UI `31038c89e18e696471f66582de54c8a298abafcc46738f92d7ba5fd196781796`.
Boot 122, Linux boot and UI PID remained unchanged. A 40-second window measured
499.944–500.044 Hz; a 2.38-second UI pause bounded MCU mixing at
498.486–500.863 Hz with 59 additional decoded and displayed frames. Observed
peaks were 803 µs mixing and 231 µs lateness. Link recoveries/errors stayed
zero and Linux reported no breaks. Decoder drops rose `0 → 26` at startup,
then remained constant; the cause was unresolved. RF remained off.

## Restoration and capture limits

Mode 0 and `stream_active=0` were restored, but the decoder retained 1280×720.
The original fallback picture was therefore not established restored. The
supplied SDK's picture-switch function sets a flag that pauses its picture
thread without an active reset path, a candidate explanation rather than proof
of installed binary behavior. No player restart was performed for that check.

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
