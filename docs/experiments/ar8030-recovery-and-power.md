# AR8030 candidate recovery and low-power acquisition experiments

[Experiments](README.md) · [Power control](../firmware/ar8030-power-control.md) ·
[Link management](../protocols/ar8030-link-management.md)

Gnd485 18.21.10 trials 143–161, recorded on October 14, 2026, separate three
results: automatic recovery after host candidate removal, native factory-offset
fallback, and an unresolved failure to acquire with the backend's initial power
level. These were temporary RAM experiments through the original Linux stack.
They precede the later [Open8030 release bench](open8030-validation.md).

The matched stock runtime is SHA256
`3e454131a334ec422c0b2e8ae4924d6bda68cb61b70fd754da26412578b39c87`,
from container
`dc58247910af564376350404383c5c411cbec21663873fba24e70c61a3584e3c`.
Power levels below are calibration indices and reported state, not measured
radiated dBm. Link state, local TX completion, host delivery and viewed camera
output remain distinct evidence.

## Candidate removal: one timeout, then two successful cycles

Trial 143 initially displayed native camera video, but removing the active peer
from the candidate list timed out during shutdown with `-110`. Re-admission
returned `-11`. Its first-version diagnostics could not distinguish engine-idle
waiting from RX-DMA completion waiting. Initial video therefore did not establish
automatic recovery.

The next implementation drained pending RX DMA before resetting its producer.
A synthetic case made completion depend on keeping the producer out of reset.
This verified the intended ordering, without proving why trial 143 timed out.
The successor also added separate stop-stage diagnostics.

Trial 144 sent an empty candidate list at 25 and 80 seconds, restoring the
original list at 35 and 90 seconds. It sent no host stop/start or deinit/init.
Socket byte counters stayed stable while the peer was excluded. Linked state
returned at samples 36 and 91 seconds; MCS 10 returned by 50 and 105 seconds.
Final diagnostics reported two recovery attempts, two completed rebuilds,
zero failures, zero spontaneous FCH losses/invalidations and zero intermediate
RX resets. One owned TX buffer was cancelled.

**Neither shutdown had RX DMA pending.** The test establishes two same-peer
candidate-removal/readmission cycles. It does not physically validate the new
DMA ordering, isolate the earlier timeout, or establish recovery from peer power
loss, token changes, another peer or a stuck engine.

All 46,956,518 offered video bytes and 334,430 control bytes reached their host
endpoints, with no queued remainder or endpoint drops in the final sample.
Three camera captures were viewed, with native identity checked around them:

| Measured interval | Valid grabber frames | Measured startup black frames |
| --- | ---: | ---: |
| First 20 seconds | 601 | 49 |
| Second 20 seconds | 601 | 8 |
| Final 50 seconds | 1,501 | 6 |

All used 1920×1080 YUYV at nominal 30 fps, with three-second warmup and no
measured errors, gaps or short frames. Grabber cadence and visible overlays do
not establish RF/camera cadence or telemetry correctness. A separate full stock
reboot, 24 getters and viewed 301-frame nonblack stock capture passed.

The failed trial-143 runtime was 54,984 bytes, SHA256
`f2cc7021ff312f7bead79253cda718517bf32a04ef8e09de1609ff8a21ad4da2`.
Trial 144 used 55,040 bytes,
`065f234f72e2e020d731bcc59787f5c00934b4ae8b458506ffbde090b8eecdcf`.
Those identities preserve the distinction between the failed and accepted tests.

## Factory fallback with working video, trial 145

Normal boot identified flash bytes `c46015c46015` and selected a 2 MiB read
bound. Both partition windows failed validation with `-74`; boot selected zero
path offsets and applied them to the initial user-0 plan. Stock getter
`0x01000071` also returned zero path offsets after restoration. This establishes
matching selected values, not a successful factory-record read or the cause of
partition rejection. The [factory-storage study](../firmware/ar8030-factory-storage.md)
describes the decoder, fallback and transport boundaries.

The 60,592-byte runtime hash was
`f65af4d20c22e5c55273dfe76bb4be01d87b7a8de4e8d4a1392827739daa1bdc`.
Its native 55-second capture had 1,651 nonblack frames and viewed camera output.
The final video counters were 21,698,889 offered, 21,697,877 delivered and 1,012
still queued, with zero endpoint drops. A queue remainder is not lost data or
proof that every offered byte had already reached the host.

This build did not implement the dynamic host power setter. Stock reported
user 0 at level 14, while the replacement retained its initial setting. Its
successful video must not be relabeled as dynamic-power parity. Recovery
artifacts taken before the verified full reboot were excluded from the final
stock-recovery evidence despite their caller-supplied labels.

## Native levels 11, 14 and 25: trials 146–148

Trial 146 integrated host setters and all-user initialization. The unchanged
backend requested `{user=0, level=11}`. The observer incorrectly expected 14
and stopped before issuing lifecycle commands. Supplemental observations showed
no endpoint payload, repeated FCH loss and disconnected UI. The setter executed;
normal acquisition did not pass.

Trial 147 added separate reported/applied power diagnostics. User 0 was 11,
users 1–11 were 25, and all 70 samples had no endpoint payload. The last sample
had 32 completed FCH-loss recoveries and zero matching login-sync requests.
That does not implicate rejection of a login power-sync flag. All 1,651 captured
frames were nonblack, yet the viewed image was disconnected UI: nonblack frames
alone do not prove video reception. A planned level-14 follow-up stopped at its
deadline guard before connecting or setting power and supplies no RF evidence.

Trial 148 reused the exact trial-147 runtime and reserved comparison windows:

| Action | Subsequent observation | Limit |
| --- | --- | --- |
| Set 14 at 20 s | No link or payload | A successful setter is not acquisition |
| Set 25 at 55 s | Linked at sample 57; camera returned | Backend immediately selected 14; later capture was predominantly at 14 |
| Set 11 at 90 s | Drop at 93, reacquisition at 95, backend selected 14 at 96 | No sustained level-11 measurement |

The final sample was linked at MCS 8, with 13,537,228 video and 155,092 control
bytes delivered and zero endpoint drops. Two zero-flag login requests matched;
none was rejected. The first two viewed captures showed disconnected UI and
the later two showed camera video. A level-25 step preceded acquisition, but
this did not identify the defect or establish a correct replacement policy.

## Stock acquired at 11; native readback alone did not explain the failure

Trial 149 loaded the exact stock image through the same bounded RAM path and
sent no observer setters. At 4.0 seconds stock had level 11 with gain fields
`[37,30,37,30]`. At 7.8 seconds it was linked while still at 11; at 8.0 seconds
the backend selected 14, with `[34,39,34,39]`. The viewed startup capture showed
the camera. These sequential 200-ms samples are not a continuous or atomic
trace, but they rule out treating level 11 as inherently unable to acquire in
this setup. Trial 153 repeated stock acquisition at 11 before switching to 14.

| Native trial | Added observation or change | Result |
| --- | --- | --- |
| 150 | Paged RX diagnostics, 280 successful page requests | 52 retained CRC-valid acquisition reports, no login extensions; 34 FCH-loss recoveries and no payload |
| 151 | Stock-compatible GET_REG readback | Gain fields matched stock at 11; no acquisition |
| 152 | Six board TX controls for frequency subranges, with change-only caching | 53 retained valid acquisition reports, no login extensions |
| 154 | Full baseband readback without changing native firmware | Twelve 5,770-MHz samples matched RX8/TX0 PLL bytes and broadcast channel 24; still no link |

Trials 151/152 each completed 490 GET_REG and 280 RX-page requests. Local TX
errors were zero in the sampled reports, but completion and buffered CRC-valid
reports do not prove that the AP received them. RX counters restart on recovery;
summing sampled counters would invent totals. Separate pages/register reads can
straddle recovery and are not atomic snapshots.

One trial-154 sample retained reports while later reads showed unloaded
compensation. Even equal recovery counters did not establish the order of
transmission and compensation changes. The older stock full-register reference
also used level 25; its live gain/status differences were not a missing-write
specification for level 11.

Two static leads also needed their original mode gates. The duration-adjustment
branch at `0x227d58` requires raw user 8 and a true result from `0x220ba8`.
That helper requires synchronization enabled at `0x263bff` and master disabled
at `0x263c00`. The matched image's four initial bytes there are `00 01 01 00`,
and its DEV loader path skips the AP synchronization-mode configuration. The
matched JSON has role `dev` and no basic `sync_mode` key. This narrows the
clock-adjustment hypothesis; static defaults are not a live RAM measurement
or proof that no other path changes them.

The inspected successful calibration path calls `0x21d764`, whose small wrapper
holds bit 0 of `BB+0x800`. That return path did not establish an additional
missing calibration-stop write. Differences at registers `0x04ec` and `0x0014`
also retained unknown semantics, so they were observations to investigate,
not values to copy into initialization. Neither static lead identified the
acquisition defect.

## Compensation ordering improved without solving acquisition

Trial 155 added event diagnostics and retained a pre-encode, submission and
local completion before compensation loading. The pre-encode readback had the
enable bit clear and zero calibration bytes. Native recovery had cleared
compensation on every context rebuild.

Original preparation at `0x223128` compares the requested bank with cache
`0x268378`, initialized to 3. The inspected mode-0 engine-reset paths preserve
that cache. The separate prepare-handler TX-release branch must not be confused
with admission's immediate hold/reset/release helper.

Trial 156 preserved compensation across normal contexts in the supported
single-calibrated-band mode. Probe/default loading/calibration and transitions
to or from bounded diagnostics still invalidated it. Hardware recorded one load
across 35 contexts and 2,958 submissions, with no software-unloaded submissions
and matching retained pre-encode readbacks. It still failed to acquire at 11.
The maximum 2,978 µs acquisition poll gap measures CPU service spacing, not IRQ
edge latency or on-air timing. There was no continuous hardware-register trace.

## Local policy and peer adaptation remained separately qualified

Trial 157 accepted selector 2 of `0x02000009`, storing 11 as all four local
initial/reset levels. It left current gains unchanged: user 0 remained 11 and
the other users remained 25. No connected-loss reset occurred because there
was no connection. It recorded 3,125 submissions without unloaded events and
still failed acquisition. Automatic/manual selector 3 and disconnect reset
had synthetic coverage only in this trial.

An unsupported backend request was traced to selector 0. Its meaning was
corrected to disabling peer adaptation, rather than immediately transmitting
remote synchronization. Trial 158 integrated selectors 0/1 and the shared FCH
transport, and verified backend selector 0 plus local selector 2 without the
previous auto-power error. With no connection, zero peer-quality samples and
queue attempts were expected. Enabled adaptation and delivery of peer requests
were not physically validated. The trial still failed acquisition at 11.

The subsequent incoming-command-8/subtype-6 checkpoint added local automatic
power handling and offline comparisons; that checkpoint explicitly preceded any
hardware trial 159. Later synchronization and channel-policy checkpoints must
be assessed separately. The later stages 188–190
[dynamic-power investigation](../firmware/ar8030-power-control.md#failed-dynamic-integration)
also retained a failed level-11 integration; successful calculation comparisons
did not establish normal runtime policy.

## TX service and two calibration corrections, trials 159–161

Trial 159 ran incoming local/peer-policy handling plus a read-only TX-service
diagnostic. It recorded 3,102 submissions/local completions across 34 contexts,
33 recoveries and 55 retained CRC-valid acquisition reports, with no login
extension or endpoint payload. No connected/incoming policy update occurred.

The diagnostic `0x0c5a002c`, daemon alias `0x015a002c`, is version 1, 240 bytes.
It retains an eight-submission ring, cumulative counters and maxima. Across
464 unique retained records, building took 199–200 µs, submission 14 µs and
total CPU service 239 µs; maxima were 212/14/264 µs. Seven TX-end handlers saw
pre-encode pending, and 3,095 terminal observations saw TX-end pending. These
instrumented CPU observations neither identify IRQ edge times nor establish
which packet an interrupt acknowledges. Rare overlaps did not explain the
persistent acquisition failure.

Trial 160 changed only pre-calibration user preparation. Original
`0x224b2a..0x224b36` sets bit 3 of `BB+0x9a0+8*user` for users 0–10 before
readiness pulses/calibration. The reset profile had these bits clear; an older
all-`0x5a` test fixture had accidentally set them and hidden the omission.
The corrected test cleared only bit 3, preserving unrelated fixture fields.
User 11 and neighboring bits remained unchanged. The physical purpose of the
bit was not independently established.

That ordering correction still produced no link: 2,907 submissions/local
completions, 33 contexts, 32 recoveries and 51 retained valid reports. One initial
submission preceded software compensation loading; the loaded bank survived
later retries. Per-boot compensation differences were not a causal result.

Trial 161 then corrected calibration convergence selection to history-ring
entry 0, following original `0x224f68..0x224f6e`, instead of the newest sample.
It passed 32 synthetic calibration cases and the recorded regression failed
against the old implementation. Hardware still failed acquisition: 3,113
submissions/local completions across 34 contexts, 33 recoveries, 55 retained
valid reports and no login extension, payload or sampled TX error.

All three native captures showed disconnected UI despite 1,651 nonblack frames;
restored stock captures showed camera video with 301 frames. Each retained four
warmup errors separately from the measured interval. Trial 161 initially had
six link-info getter failures (`-1`), then all 24 passed after settling. Stock
user 0 was 14 and users 1–11 were 11 after trials 160/161, differing from earlier
recoveries where the other users were 25. That lifecycle variation remained
unexplained and was not erased from the recovery record.

Trial 161's 68,540-byte runtime was
`ad8fdf8649378a2ac37685b23e0cc691578c6624b71ec0bf973e3666b94d4d78`.
Subsequent [channel and synchronization work](../protocols/ar8030-channel-integration.md)
initially had offline coverage. Its final frame-duration build later failed
physical trial 162. [Trial 163](open8030-baseline-revalidation.md) restored
camera video with the preserved stage-145 runtime, leaving the newer
implementation's acquisition regression unresolved.

## Diagnostic boundaries and review validation

The investigation added diagnostic views to distinguish these failures:

| Diagnostic | Shape and observation boundary |
| --- | --- |
| Recovery `0x015a0026`, version 2 | 84 bytes; reasons 1/2/3 are FCH loss, token/profile change and candidate removal; stop stages 1/2/3 are DMA drain, engine idle and done |
| Power `0x015a0027`, version 1 | 80 bytes; separate 12-byte reported/applied arrays, host-set/login-sync counters and session-local elapsed time |
| RX page `0x0c5a0028`, daemon alias `0x015a0028` | LE16 offset/count, count 1–1,024 within the 4,084-byte structure; success returns precisely that range, malformed input `-22` |
| TX timing `0x0c5a0029`, alias `0x015a0029` | 256 bytes: sixteen LE32 header fields and two arrays of eight three-word events; each array retains its context's first eight events |
| Local policy `0x0c5a002a`, alias `0x015a002a` | Twelve LE32 words, 48 bytes; selected levels, policy writes and connected-loss resets |

The 4,103-byte full framed RX reply had timed out in trial 146. Paging supplied
a working route without proving the precise transport failure. Native GET_REG
`0x01000064` returned fixed 256-byte replies, accepting unaligned/zero-length
reads, with `-6` for out-of-range windows or counts above 256 and a zeroed unused
tail. None of these read-only diagnostics changed RF policy by itself.

For this wiki review, 77 saved function ranges and 12,668 distinct GNU
instruction words were checked against the original runtime. The checkpoint
power planner was rebuilt and matched all 760 retained original-instruction
cases; factory selection matched 2,072 cases, including 2,052 decoded records.
The compiler/ctypes layout check and FCH-monitor ASan/UBSan tests also passed.
The original emulator, hardware experiments and camera inspections were not
rerun. Saved reports retain full-stock-reboot/getter/camera recovery separately
from each native trial's success or failure.

The subsequent handoff review checked another ten retained instruction ranges
and 7,276 instruction locations for the synchronization gate, tuning and
calibration paths, plus the four static synchronization bytes above. Their
hashes/words matched the original runtime. This verifies the retained analysis
inputs without adding original-emulator execution or physical observations.
