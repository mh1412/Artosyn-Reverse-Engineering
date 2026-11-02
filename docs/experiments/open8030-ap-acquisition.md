# Open8030 AP acquisition and first camera video

[Experiments](README.md) · [AP preparation and throughput](open8030-ap-bringup.md) ·
[Release validation](open8030-validation.md)

The October 2026 Ascent Lite Sky482 investigation progressed from digital TX
activity without reception to a native AP/DEV link, exact short test streams,
and camera video. Those were separate milestones. The AP used explicit board
preparation while DEV used the receiver boot path; sharing an executable did
not yet establish normal configuration-selected AP startup.

These are retained experimental observations and matched-image instruction
analysis. The hardware trials and raw captures were not rerun for this article.
The [preparation record](open8030-ap-bringup.md#matched-stock-input) identifies
the 18.21.10 stock inputs. Later release results have separate image identities.

## Digital TX activity without a received link

The first one-second AP session submitted 342 packets and observed 351 TX-end
interrupts, but received no reports or synchronization ACKs. A ten-second trial
with both radios running `d929648` submitted 3,451 packets and observed 3,532
TX ends. The receiver searched the selected channel without valid frame-header
indications or admission. A TX-end counter was therefore insufficient evidence
of a decodable transmission.

Several concrete initialization corrections preceded successful reception:

| Investigation | Finding and bounded result |
| --- | --- |
| Band routing | PLL shadows had been written without TX8/RX0/RX11 band selectors; correcting selectors alone still produced no receiver headers |
| Packet memory | Shared layout and post-user masks were absent after PSRAM setup; correcting these still did not establish reception |
| Polling command cursor | Missed preparation edges left the cursor behind; reanchoring an unambiguous raw-user/direction preparation reduced errors from hundreds to one initial body event, without restoring reception |
| Path B | Reproducing the matched host's path-disable setting changed the expected registers but did not establish reception |
| Unused context 10 | Source-backed RX/TX holds corrected an initialization gap without restoring the link |
| Closed-loop power mode | Selecting both mode bits across twelve contexts corrected an omission; user-8 readback changed from `0x3c300000` to `0x3cb00000`, but reception still failed |

Separate experiments moved the control/header DMA source, changed retry control,
tuned paired directions, copied the observed clock setup, added calibration
context bits, and initialized unused normal contexts. None established reception;
those speculative changes were reverted. The retained power-mode correction
made inputs consistent and did not constitute a measured power-control result.

Initial scan scores could not diagnose missing RF: while disconnected, the
normal scan had advancing tunes but no armed measurements. A receive-only
survey then produced samples independently of login. Its first stock on/off
extrema overlapped; that comparison did not validate an RF detector.

A later fixed-frequency 5,770-MHz comparison retained only samples whose
measurement count advanced, excluding transition windows:

| Condition | Samples | Mean raw reading | 90th percentile |
| --- | ---: | ---: | ---: |
| Stock running | 168 | 108,726 | 211,343 |
| Stock stopped | 354 | 51,191 | 54,818 |
| Native AP held | 176 | 51,109 | 54,288 |
| Native AP running | 354 | 51,248 | 54,422 |

This supported investigating RF output and routing before further login changes.
The units were uncalibrated; the result did not prove zero radiation, identify
the spectrum, or measure emitted power. Later 5,839-MHz controls required their
own baseline rather than reusing the earlier threshold. Surveying also suspended
normal login, so missing reports during those windows were not an independent
acquisition failure.

## Rejecting misleading register and function-name clues

Matched instruction tracing explained two apparent analog-control differences:
project subcommand 138 selected ADC measurement by changing the top byte of
`BB+0x08c` and bit 16 of `BB+0x090`; subcommand 139 stopped that service. The
stock/native difference was consistent with stock ADC channel 0 being enabled,
not a demonstrated missing RF-enable operation.

The matching host's `fpv_bb_power_bypass_extPa` name was another misleading clue.
Its project subcommand 15 reached a success return in this radio image without
applying the argument or writing registers. That does not describe every vendor
build. The separate path-B operation did perform writes.

The packet worker used the same segment writer for AP and DEV: eight-byte budget
rounding, MAC/outband/stream segments, then the descriptor-list terminal pulse.
Static callback and task-table tracing did not find an unconditional missing
TX enable in the examined TX-EOP path. Indirect subscribers and queued consumers
were not thereby excluded.

Budget-change helper `bb_phy_mcs_tx_set_end` performed MCS bookkeeping and event
publication, rather than direct RF programming. Event 9 advertised twelve bytes
while its producer explicitly initialized nine. Reproducing undefined padding
would not establish protocol compatibility.

## C/D pin assignments precede the first native link

The initial board comparison omitted three mux words. Extending the diagnostic
exposed these differences against a stock RAM observer:

| Register | Stock acquisition | Earlier native AP |
| --- | --- | --- |
| `0x0a10a034` | `0x00422622` | `0x00000000` |
| `0x0a10a038` | `0x11112222` | `0x11112222` |
| `0x0a10a03c` | `0x00077011` | `0x00022011` |
| `0x0a090140` | `0x00000228` | `0x00000228` |

Matched startup and function-table analysis supplied eight additional records:
C0–5 and D11–12. The pin API was extended beyond A/B, validated the entire batch
before writes, and preserved unlisted fields. Applying those records reproduced
the two differing mux words and established the first native AP/DEV link with
the preceding RF settings and gains.

This implicates omitted pin assignments in the earlier failure, but does not
identify each signal's electrical function or isolate which pin is necessary.
No whole mux register was blindly copied, and the DEV boot profile was unchanged.

The ten-second trial accepted 1,724 receiver reports and one synchronization
ACK. AP submitted 10,213 generated application bytes and received 233; DEV
recorded 1,727 FCH indications and 3,197 MAC packets after stock traffic ceased.
Shutdown returned to prepared state without error. Those counters did not
compare complete application contents, and no camera fed the native AP yet.

The AP container was 101,888 bytes, SHA-256
`4d3bedc1c0194893b7ee257a511ee377f01849dbe97f3a3207728ad3eb02ffd3`;
runtime SHA-256
`a76c6ee26afc6d5edfdb7a5ddd7a644734305ac7dc088df17035b052cbe82410`.
The receiver still ran `d929648`, so this was not a same-revision test.

## Exact short streams and lifecycle were separate checks

Initial managed-stream captures delivered 9,956 of 10,213 downlink bytes and
985 of 1,021 uplink bytes, all in expected order. Zero receiver endpoint drops
did not account for the missing bytes. Quiet AP capture then received the entire
downlink, but still lost the first 36 uplink bytes. Different polling and startup
histories prevented attributing that improvement solely to diagnostic workload.

With the same optimized container on both radios, a later trial received all
10,213 downlink and 1,021 uplink bytes exactly. Its first session failed login;
the second connected after a clean stop and two-second pause. Container SHA-256:
`ee2a26dad7eade53b0b15773c18698bfe463ac023e2e11f387d8a927a7880542`.
Compiler settings and receiver revision changed together. A subsequent lifecycle
candidate again lost the 36-byte uplink prefix, so the exact trial was not a
general loss fix or reliable first-attempt acquisition result.

Explicit AP preparation later bridged into normal group-`0x0b` lifecycle commands.
Held configuration saved a peer/session plan without starting RF; continuous
start, stop, deinit, init and restart passed one hardware sequence, including
operation beyond the earlier ten-second diagnostic bound. Stop drained pending
RX DMA before releasing TX storage; a failed handshake retained ownership.
That lifecycle trial passed downlink but failed complete uplink delivery.

The early development command formats were:

| Direct project ID | Payload | Scope |
| --- | --- | --- |
| `0x0c5a0030` | `AP01`, 16 bytes | Frequency, local identity and resolved gain/amplitude inputs; held preparation |
| `0x0c5a0030` | `AP02`, 20 bytes | Adds path-B-disable flag at byte 16 and reserved zeros |
| `0x0c5a0031` | `APS1`, 16 bytes | Explicit bounded start, duration 1–10,000 ms |
| `0x0c5a0034` | `APS1`, 16 bytes, duration zero | Save held host-session configuration; does not start RF |
| `0x0c5a0034` | `APH2`, 116 bytes | Adds target internal MCS at byte 15 and 25 LE32 FCH keys |

The configured broadcast started at acquisition index 1 and switched its target
after a connected TX-end, outside active packet ownership. This was fixed coding,
not automatic adaptation. Later [AP03/AP04 profiles](open8030-ap-bringup.md#fixed-experimental-profiles)
added width and length selection. Diagnostic layouts evolved with the checkpoint.

## Cooperative work and the first camera pipeline

Higher coding initially delivered only small fractions of generated streams.
Preparing a separate immutable packet ahead of pre-encode, removing redundant
copies, reducing checksum work, and polling RF during host admission improved
short tests. Missing bytes and uplink losses persisted; faster preparation was
not a complete-transfer pass. Enabling instruction cache reduced observed maximum
AP preparation from about 1,119 to 306 microseconds. Data cache remained disabled
at this stage; later cache-enabled integration is a separate implementation.

Keeping the camera process stopped across daemon replacement did not reopen its
SDK sockets. Starting it again after normal boot encountered media initialization
problems. The successful experiment instead loaded native AP before the camera's
first initialization in that Linux boot, using the stock one-shot maintenance
startup. That path loaded media drivers and updater without starting the camera
or radio stack; unlike the separately studied VRX hook, it did not source an
arbitrary script named in the selector.

The first maintenance trial drove Linux GPIO62 low and failed login. Restoring
its direction to the observed normal-startup input state, together with normal
LED setup, preceded connection and camera writes. This supports a GPIO setup
dependency but does not isolate its electrical function or failure rate.

Both radios used the instruction-cache container SHA-256
`c6bcc3b2374a1c37d3ef34abac576fa7461c36c49e7da834a1263ce166af4a47`.
A 180-second recording had 179 paired samples over 179.24 seconds:

| Observation | Result |
| --- | ---: |
| Connected pairs | 179/179 |
| Additional receiver video bytes | 3,836,485 |
| Additional receiver host-data bytes | 47,971 |
| Additional AP return host-data bytes | 1,144 |
| Additional endpoint drops | 0 |

Inspected images and ongoing camera socket traffic supported native AP camera
delivery. The requested hand-motion challenge was unconfirmed. The picture had
artifacts and roughly 0.2-Mbit/s displayed bitrate. Capture cadence and displayed
delay were not independent RF or latency measurements. Separately sampled
fragments contained 79 unique checksum-valid transmitter-status envelopes and no
FC/MSP envelopes; this did not prove absence of all FC traffic. Voltage remained
zero while its ADC request was unsupported.

## Rate reporting improved camera policy, not proof of a radio ceiling

The SKY backend queried `0x01000006` with direction/slot `{0,0}`, reading internal
MCS from byte 0 and LE32 kbps from bytes 4–7. Unsupported native replies left its
throughput at zero. A narrow AP TX implementation reported scheduled capacity:

```text
floor((align_down(packet_budget, 8) - 60) * 8000 / frame_us)
```

The reserve covered control, framing and a payload descriptor. Disconnected or
stale-budget states reported zero; a late subscription could receive a current
rate snapshot. This was an integration policy, not full vendor PHY-rate parity.

One later 44.522-second session reported 2,357 kbps, with the backend confirming
that value and encoder-policy MCS 8. It delivered 9,264,055 video bytes, about
1.665 Mbit/s, with visible camera video. Earlier attempts with both the new and
previous images failed login, so adding diagnostics did not establish a startup
fix. The matched Linux encoder policy also capped its target at 20,000 kbps.

Later counters measured roughly 2,817 microseconds between pre-encode events,
half the original duration assumption. Exact 3.904-Mbit/s generated delivery
exceeded the reported 2,357 kbps. The early formula's time conversion was therefore
provisional, not a physical ceiling. [Subsequent transport tests](open8030-ap-bringup.md)
separate that correction, wider profiles, DMA failures and later short throughput
results from camera quality and sustained release acceptance.
