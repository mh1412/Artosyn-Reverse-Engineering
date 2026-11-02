# AR8030 host delivery, PHY synchronization and first decoded video

[Experiments](README.md) · [Uplink and login](ar8030-uplink-and-login.md)
· [Original-backend integration](open8030-early-dev.md)

Gnd485 18.21.10 DEV trials 108–126, recorded on 14 October 2026 UTC, progress
from host-delivered RF bytes to offline-decoded video and real native status/events.
The stock daemon and backend were stopped during these private probes. Later
trials 127–135 establish video through that original host stack; offline decode
here must not be described as that later end-to-end result.

## Receive sockets delivered bytes before video could decode

Trial 108, operation 63, opened host socket 3 with a 16 KiB receive buffer and
drained it during a ten-second radio monitor. All 38,552 observed MAC-port-4
bytes reached the Linux probe in 400 stock-format messages. Endpoint drops,
remaining queued bytes and malformed host messages were zero. There were 516
valid MAC/control packets, 145 TX completions and 57 request-ID recovery cycles.

Forwarding occurred only after control processing could invalidate the peer or
token. Bytes were copied before releasing the DMA slot, then committed only
after SDIO copied them into its own queue. One SDIO entry remained reserved
for request replies. This accounted for local delivery, not missing RF packets.

The stream contained four vendor metadata blocks reporting 1920×1080/60, two
distinct frame IDs and three retained type-1 HEVC NALs. It contained no VPS,
SPS or PPS. The conservative deframer omitted the final unterminated Annex-B
unit. `ffprobe` could not establish dimensions or decoded frames. Metadata
dimensions and a few slice headers did not establish usable video.

Trial 107's preceding instruction work had tested exact encoding `ad8c0c77`
with register s8 as input/output against adjacent-byte swaps within each 16-bit
lane for 1,024 random hardware inputs; the previously tested a5 encoding
`ad8787f7` passed again. That enabled a larger socket-thread reconstruction.
It did not identify the CPU or validate every encoding of that operation.

## Measurements, report status and a register no-op

Operation 64 added logical-RX-user-8 end-of-packet measurements. Raw user 9
aliases 8; a per-user guard suppresses duplicate EOPs until the next FCH event.
The helper samples error/SNR/gain/offset fields while preserving bank selectors.
If hardware FCH validity is clear, a zero raw error becomes one. Native code
zeroed bytes 9–11 of the 16-byte sample where stock left scratch residue.
These were raw observations, not independently calibrated RF measurements.

Trial 109 additionally tested exact instruction `8204140b` at `0x22d28a`, using
s0, against full 32-bit byte reversal for 1,024 random hardware inputs. The
partial ISA model was extended only for that encoding. Generic analysis had
failed to recover the EOP procedure; seeded exports and original instructions
supplied the evidence.

| Trial | Local / duplicate EOPs | Forced-error samples | Host video bytes | Recovery disable / enable |
| --- | ---: | ---: | ---: | ---: |
| 109 | 537 / 537 | 204 | 38,033 | 57 / 56 |
| 110 | 543 / 542 | 314 | 39,212 | 57 / 57 |
| 111 | Separate post-login buffer probe | — | 35,516 | 56 / 56 |

The configuration disabled outbound DEV `sq_report` while enabling AP reports.
Lack of native outbound command-2 measurements therefore did not establish a
deviation from that profile. Local samples still mattered to PHY policy.
Both LNAs were manual/on; trial 110 changed only the two report-plan status
bytes from 0 to 2. The high report bits then matched stock, while the already
corroborated analog controls stayed unchanged. Throughput did not recover.

Trial 111, operation 65, tested a supposedly missing buffer transition after
login. Readbacks were `BF0: 41410101 → 41410101` and `D08: 0 → 0`—no change.
Admission already applied that basic mode. Repeating this no-op was not a
throughput fix, and none of these captures gained parameter sets or decoded frames.

## Host transmit and a remote keyframe ACK

Trial 112, operation 66, added copy-mode TX sockets and the backend's requested
56,320-byte video RX capacity. It opened host ports 3 and 2 with RX+TX, each
with 2,048-byte TX capacity; control RX was 2,048 bytes. The native endpoints
supported up to 4,096 TX bytes and two pending writes per port. Incremental
copying, fair service and retryable completions were cooperative approximations
of stock workers. The then-immediate close behavior was later superseded by
[deferred close](../firmware/ar8030-tx-and-stream-ownership.md#stock-workers-versus-cooperative-close).

Host payload could enter MAC only in state 2/slot 0. A pending synchronization
ACK had priority even when its current budget was insufficient. At most one
132-byte host fragment fit each copied MAC packet. Stream positions, immediate
enqueue replies and later write completions followed the
[matched host contract](../protocols/ar8030-host-link.md#two-stage-write-completion).

The probe sent a 13-byte application message: subchannel 0, sequence 1,
command `0x20`, value 1. The retained 41-byte MAC packet contained those exact
bytes. A 12-byte CRC-valid reply appeared at control-stream offset 1,056,
matching subchannel and sequence, setting ACK and returning command `0x20`.
Original backend dispatch routes that flag to its pending-message ACK handler.
This supports remote application receipt of this request, not keyframe production.

Trial 112 delivered 31,567 video and 5,777 control bytes with no endpoint drops.
There were 93 CRC-valid application envelopes, including the matching ACK, and
one invalid candidate. Video still lacked parameter sets and decoded frames.
`ffprobe` returning process status zero was not decode success. The existing
backend's repeated keyframe requests and session behavior remained separate.

## FCH identity explained the later receive-policy failure

Operation 67 integrated an eight-sample receive-MCS policy using the configured
rows 1, 2, 5, 7, 8, 10 and 12. In trial 113, only the initial MCS-1 request was
acknowledged locally because hardware already matched. No command was advertised
over FCH. The final window had raw average SNR 13 and eight bad/consecutive-bad
samples; retaining MCS 1 followed those inputs. Stock's later asynchronous
MCS-12 samples were not an acquisition-matched control. Forcing a higher MCS
would not have resolved the unexplained measurement difference.

Trial 114 added trace and aggregate buckets. It showed that selected and
duplicate EOPs had similar raw SNR distributions; the final low sample was not
representative of the interval. Hardware-valid FCH stopped about 110 ms after
login while CRC-valid DMA packets continued. The retained final FCH command
`25 16 04 48 cb 6a 06` passed its check and requested an RX identity update.
Native code had validated the message without dispatching command 6.

Operation 69 implemented the supported identity callback and shared cache/timer
rules. It applied RX8/shared identity `cb` after login. Trial 115 then had 562
valid connected FCH observations and no recovery cycles; MCS moved 1→2 with
a request for 5 pending at ten seconds. Operation 70 extended only the maximum
observation to fifteen seconds:

| Trial 116 observation | Result |
| --- | --- |
| Valid MAC/control packets | 2,885; zero invalid |
| TX submissions / completions / errors | 1,447 / 1,447 / 0 |
| Connected valid / invalid FCH | 1,444 / 0 |
| Recovery disable / enable cycles | 0 / 0 |
| MCS requests / ACKs | 6 / 6; initial ACK local |
| Advertised requests / cancellations | 5 / 5 |
| Observed MCS | 1, 2, 5, 7, 8, 10 |
| Video / control bytes delivered | 39,825 / 21,158; no endpoint drops |

The control capture retained only 8,192 bytes. Its truncation was not endpoint
loss; trial 114 similarly retained 8,192 of 8,536 delivered bytes. Trial 116
still contained only two video frame IDs, no parameter sets and no decoded video.
Longer observation and real MCS negotiation therefore did not solve the session
problem. Its 256 unsupported callback dispatches lacked a per-command histogram;
their identities could not be inferred from that aggregate.

## TX MCS and the missing video-sync sequence

Trial 117 captured 250 command-3 requests for TX MCS 2, while video bytes stopped
around 8.22 seconds and control continued to fifteen seconds. No accepted
command 1, 2 or 4 was observed. Operation 72 implemented the supported upward
TX change; trial 118 applied MCS 2 and the AP stopped repeating that request.
Video still delivered only 40,109 bytes without parameter sets. Downward MCS
drain handling and wider modes were later work, not implemented by this trial.

The matched backend sends a separate video-sync command `0x28`, value zero,
ten times at a 300-ms timer interval after link-up. Operation 73 added that
sequence after the one keyframe request, waiting for each write completion.
Initially, first delivered control data substituted for the stock link event;
that was not full session parity. The control capture grew to 32 KiB.

Trial 119 completed all eleven writes, 143 bytes, and received the keyframe ACK
plus two sync ACKs. Video MAC payload rose to 776,159 bytes. A stale probe
assertion still expected one completion and exited with code 39 before saving
video or endpoint diagnostics. This was an observation-tool failure after the
writes, not a decode result or evidence that only one request completed.

Trial 120 repeated the operation with that probe-only assertion corrected.
All 673,063 video and 14,896 control bytes reached the host, with no endpoint
drops. The video contained 206 distinct frame IDs, nine VPS/SPS/PPS groups and
36 IDR NALs. Offline decoding produced 200 frames at 1920×1080. The historically
viewed image showed the camera scene with corruption; decoder stderr retained
missing references, invalid QP and mixed-NAL errors. Reported container frame
rates did not establish camera cadence.

Payload DMA stopped at 12,572,648 µs although valid FCH and TX continued to the
fifteen-second deadline. Nineteen recovery cycles returned under the greater
load. Poll/handler timing motivated service-latency and buffer investigations;
neither was yet established as the sole cause.

## Copy latency, endpoint overflow and clean retained suffixes

The runtime polled with interrupts disabled. Host service copied data at socket
peek, envelope construction and SDIO submission. The controls changed batch
size, polling opportunities and then the compiled copy routine:

| Trial | Host-service change | Video MAC bytes | Endpoint drops | Last payload DMA, µs | Maximum poll gap, µs |
| --- | --- | ---: | ---: | ---: | ---: |
| 120 | 4,096-byte batch | 673,063 | 0 | 12,572,648 | 5,427 |
| 121 | 493 bytes; one SDIO block with envelope | 1,591,832 | 206,212 | 14,999,898 | 2,868 |
| 122 | 1,005 bytes; two blocks | 882,814 | 0 | 13,182,609 | 3,812 |
| 123 | Two blocks, RF polls between copies | 1,703,644 | 219,045 | 14,997,502 | 2,551 |
| 124 | Same policy, aligned libc word copies | 1,696,987 | 0 | 14,998,866 | 2,373 |

Trial 121 delivered 1,385,620 bytes but retained only the first 1 MiB, which
decoded 271 frames with errors. Trial 122 delivered all bytes and decoded 234
frames, also with errors, but stopped payload reception early. Trial 123 reached
the deadline with no recovery cycles; 1,484,599 video bytes were delivered and
the retained first MiB decoded 260 frames with errors. Endpoint overflow had
become a separate bottleneck. Its append-between-peek-and-commit regression
preserved the immutable scratch copy while newly received data extended the queue.

The actual build still used bytewise `memcpy`. Trial 124 enabled Zephyr's aligned
word copy/set routines with `CONFIG_MINIMAL_LIBC_OPTIMIZE_STRING_FOR_SIZE=n`;
different source/destination alignments still used byte copies. All video and
20,985 control bytes reached the host, with no recovery cycles, invalid MACs or
TX errors. This measured improvement does not prove every previous receive stall
had the same cause.

The first-MiB capture's untrimmed deframed stream decoded 258 frames with initial
errors. An **exact suffix** beginning at its first VPS, byte 30,291, decoded 257
frames at 1920×1080 with strict error detection and empty stderr. No parameter
sets or video bytes were injected. This is a bounded clean suffix after initial
acquisition, not proof that the entire delivered stream was retained or clean.

## Status, events and the remaining host-stack boundary

Trial 125 added the matched 300-byte status getter using actual native state,
registers and the fixed-channel plan. It observed active state 2/RX MCS 1,
then stopped state 0/MCS 25. It delivered 1,650,167 video bytes without endpoint
drops; a first-VPS suffix at deframed offset 33,530 decoded 265 frames strictly.
This getter required prior probe initialization and did not yet run normal
backend startup.

Trial 126 implemented link/MCS subscriptions and a copied 16-entry notification
FIFO with explicit overflow counts. Duplicate subscription did not invent an
initial event; unsubscribe removed that source's pending messages while retaining
others' order. Notifications committed only after SDIO accepted a copy.

The probe observed four link transitions `3→0→1→2→0`, six RX-MCS transitions
`25→1→2→5→7→8→10`, and TX `1→2`. It triggered the ten-repeat sync timer from
the native linked event, replacing the control-data substitute. Event and sync
timestamps in the receipt have different origins; subtracting their first
values does not measure the initial 300-ms delay.

All 1,653,960 video and 21,081 control bytes reached the endpoints without drops;
payload reception continued to 14,995,321 µs. The exact first-VPS suffix at
deframed offset 40,087 decoded 253 frames at 1920×1080 with empty strict stderr.
There were no TX errors or recovery cycles. Candidate configuration and normal
backend-owned lifecycle remained to integrate; the probe still owned the session.

## Evidence checked for this consolidation

Radio addresses are bound to runtime SHA-256
`3e454131a334ec422c0b2e8ae4924d6bda68cb61b70fd754da26412578b39c87`
at `0x204000`. Backend addresses use SHA-256
`243c11864d266caebdc76d9c4357ca6517fd1afdbbfcbaf9d6c59c113d6d6216`,
with VA `0x400000` at file offset zero. Older backend address maps differ.
Synthetic ELF declarations, partial ISA models and Linux DWARF are analysis
inputs, not original RV32 source declarations.

This review rechecked the backend CRC32C table, exact request/ACK bytes and
capture offset, selected original socket instruction ranges, three retained
trial-116 FCH controls and the saved trial-126 event sequence. It verified that
both retained decode inputs are exact suffixes and reran strict `ffprobe`
frame counting with version 6.1.1: 257 frames for trial 124 and 253 for trial 126,
both 1920×1080 with empty stderr. The suffix SHA-256 values are respectively
`6167169120b85f80e427791c0ce1ebf2846b19516963b55a00c9414c65972d74` and
`e00adb954b499ec090e591ddc39eaa885a41ee6829c3cb8eebde8b6601cffb82`.
This did not rerun the radio trials, inspect fresh camera output, or replay all
raw timing traces and historical sanitizer suites.

Historical stock restorations used a full reboot, 24 getters and directly viewed
capture-card output. They establish stock recovery, independently of the native
offline results. The next [original-stack trials](open8030-early-dev.md) close
the host-integration boundary with fresh output while the native radio runs.
