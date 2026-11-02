# Open8030 early receiver integration and lifecycle trials

[Experiments](README.md) · [Boot and lifecycle](../firmware/ar8030-boot-and-lifecycle.md)

## What these trials establish

The October 14, 2026 records show native AR8030 receiver firmware delivering
camera video through the original Gnd485 18.21.10 Linux daemon, backend and UI.
They progress from a bounded initialization probe to firmware-owned preparation,
a source bootstrap, continuous timing, ordinary loader JSON/OTP identity and
explicit host stop/start/reinitialization.

These were **temporary RAM trials on one matched DEV board/profile**, followed
by stock restoration. They are distinct from the later
[AP/DEV release bench](open8030-validation.md). Historical build sizes, unsupported
commands and temporary packaging do not describe that later release.

Earlier [receive acquisition](ar8030-receive-acquisition.md) and
[uplink/login trials](ar8030-uplink-and-login.md) establish the preceding radio
milestones. Local TX completion, a matched AP assignment and command-stream
parsing all preceded the Linux/backend video integration described here.
The [first-video investigation](ar8030-first-video.md) additionally preserves
remote application ACKs, offline decoding and native status/events as distinct
prerequisites to the original-stack result.

This review read seven reconstruction narratives, their selected saved validation
receipts and two build/profile manifests. Fourteen distinct artifact identities
matched fifteen referenced hash/size checks; five stock worker descriptors and
the JSON-overlay arithmetic were independently checked. No hardware trial,
image inspection or raw capture reanalysis was performed anew. Direct viewing
and camera-path assertions below are recorded observations from those trials.

## Original-stack ownership and failed integration attempts

The stock backend owned both video/control sockets and supplied candidates and
session commands. A separate TCP observer read identity/status/counters without
opening competing sockets. Fresh backend/decoder startup, advancing native
delivery, identity replies bracketing the capture and a viewed image together
supported the camera-path result. A capture label or changing pixel fingerprint
alone would not identify the running radio firmware.

| Trial | Finding or failure |
| --- | --- |
| 127 | Native RF preparation succeeded, but HELLO advertised zero logical devices; backend video-buffer configuration also failed on restart |
| 128–130 | Cleanup attempts stopped before native loading: unresolved media symbol, VB cleanup failure, then inadequate process-exit delay |
| 131 | Reloading unchanged stock media drivers enabled backend sockets, candidates and link/MCS events; observer still wrongly rejected positive device count |
| 132 | Device-count handling corrected, but the daemon consumed diagnostic group `0x0c` locally; late capture overlapped shutdown and did not prove fresh video |
| 133 | Read-only diagnostic aliases enabled full-window identity/counter observation and viewed camera output; endpoint dropped about 2% of offered video bytes |
| 134 | In-place host-frame encoding removed a redundant copy; the short trial delivered all offered video bytes without endpoint drops |
| 135 | Same runtime, longer 60-second RF window and 55-second measured capture; deadline shutdown and complete stock recovery passed |

Advertising one logical physical radio at HELLO's LE16 +2 allowed the stock
daemon to enumerate it; the UART diagnostic build could still advertise zero.
The project aliases under `0x015a0000` exposed only identity/counters, not
initialization or RF-start writes. Direct SDIO commands retained their own IDs.

The original daemon's 4096-byte encoder buffer leaves **4077 payload bytes**
after framing. A complete 4084-byte RX diagnostic exceeded it and timed out;
that did not invalidate smaller diagnostics or the already completed video
capture. Later [segmented diagnostics](../firmware/open8030-runtime.md) address
this independent host limit.

The in-place encoder skipped a second copy only when payload already occupied
the final wire-frame region. Endpoint storage remained owned until SDIO accepted
its copy, with RF polling between queue access, framing and submission. Those
early transfers used at most 1005 payload bytes/two SDIO blocks. Later larger
transport limits must not be substituted into the trial's performance analysis.

## Bounded video and autonomous initialization

| Trial | Main change | Delivered video bytes | Endpoint video drops | Measured capture frames / startup black frames |
| --- | --- | ---: | ---: | --- |
| 133 | First observed full original-stack path | 14,078,458 of 14,385,211 offered | 303,392 bytes; 3,361 queued | 1,051 frames over 35 s; black count not summarized here |
| 134 | Removed redundant framing copy | 14,247,185, all offered | 0, no final queue | Separate 35 s camera capture |
| 135 | Extended bounded observation | 20,799,230, all offered | 0, no final queue | 1,651 / 43 over 55 s |
| 136 | RF initialization moved into firmware | 21,190,779, all offered | 0, no final queue | 1,651 / 41 over 55 s |
| 138 | Replaced stock bootstrap with source bootstrap | 21,124,102 | 0, no final queue reported | 1,651 / 37 over 55 s |

Trial 135 also completed 23 control writes totaling 383 bytes and delivered
142,180 control bytes. Its maximum observed RF poll gap was 3861 µs. The radio
received video through 59,999,175 µs, then returned from linked state 2/MCS 10
to idle 0/MCS 25 at the bound. Zero endpoint drops describe that delivery queue
and observation window; they are not proof of lossless RF.

Trial 136 eliminated the separately staged initialization probe. Its boot state
reported version 1, stage 8, error 0, 5632 default-data bytes and a 60,000 ms bound.
The normal backend's candidate command still started RF. It retained compiled
effective settings and the stock bootstrap; autonomous initialization did not yet
mean ordinary JSON consumption or a wholly replaced boot chain.

Trial 137 omitted the bootstrap and remained at ROM identity. Full stock recovery
passed. Trial 138 used the 792-byte source bootstrap with the same autonomous
runtime as 136, observed runtime identity and camera video, then restored stock.
The offline 15-case bootstrap store comparison supports a narrower statement
than full ROM/timing equivalence.

## Continuous session across timer wrap

Trial 139 used a continuous firmware session inside a **200-second externally
bounded observation**, with fallback at 255 seconds and a 180-second measured
capture. Continuous firmware operation did not remove the harness deadline.

The report contains 190 one-second samples and video continuing across 60,
107, 150 and 180 seconds. Final link/MCS remained 2/10. It delivered 75,297,804
video bytes with zero endpoint drops and 1334 bytes still queued while streaming;
control delivered 450,645 bytes with no final queue or drops. The trace reached
1,022,510 linked polls, maximum poll gap 4066 µs and maximum handler time 2099 µs.

The last video diagnostic was 186,601,942 µs and MCS sample time 186,593 ms.
The capture contained 5402 valid frames and 41 measured black startup frames,
at about 30.00468 capture fps. This supports operation beyond the old one-million-
poll/60-second limits and through the 107.374-second timer low-word wrap. It
does not establish indefinite uptime, physical peer-loss recovery or all later
diagnostic/millisecond wraps tested synthetically.

## Loader failure and corrected normal configuration

Trial 140 failed before RF with configuration error −22. The loader overwrote a
longer embedded JSON string with an unterminated 10,341-byte external file,
leaving old text before the embedded terminator. Its standalone stock recovery
passed 24 getters and a viewed, nonblack 301-frame capture.

The next parser extracted one complete root object from bounded loader memory,
then validated that object. Trial 141 reported consumed length **10,340**,
CRC32C **1,625,207,254**, and identity source **2, OTP**. The
[boot/configuration article](../firmware/ar8030-boot-and-lifecycle.md) explains
the stale suffix, CRC convention and distinction from ordinary whole-span parsing.

One-second samples observed 4,915,000, 4,945,000, 5,839,000, 6,015,000,
5,065,000 and 5,770,000 kHz before linked state 2/MCS 10. These are sampled visits,
not a complete search-order trace or validation of RF quality on every frequency.
The run delivered all 22,009,667 video and 140,860 control bytes with no endpoint
drops/final queue. Its 55-second capture contained 1651 valid frames, including
51 black startup frames, and a reported viewed camera image.

The package manifest still says `boot_acceptance_verified=false`: it was a
packaging receipt, generated before hardware acceptance. The later trial receipt
adds boot/video evidence. Neither file should be rewritten or interpreted as
independently proving more than its own stage.

## Explicit host lifecycle

Trial 142 retained the normal JSON, hardware identity and stock backend while
the observer sent lifecycle commands through the original daemon.

| Sequence, seconds into observation | Result |
| --- | --- |
| Stop 25, start 30 | Video/control counters unchanged during pause; linked by 31, MCS 10 by 40 |
| Stop 80, deinit 81, init 82, start 85 | Counters unchanged during pause; status unavailable while deinitialized; linked by 86, MCS 10 by 95 |
| Duplicate stop/start/init | Returned −7 as required by the native state contract |

The final lifecycle diagnostic reported version 1, running state 2, three starts,
two stops, one init, one deinit, no cancelled RX/TX ownership and no error.
That zero-cancellation result means the hardware run did **not** exercise stuck
DMA, forced ownership cancellation or late completion after timeout.

Final video accounting was 52,014,120 offered/accepted, 52,013,073 delivered and
1047 queued, with zero endpoint drops; all 351,839 control bytes were delivered.
Initial/restart/reinit captures contained 601/601/1651 valid frames over
20/20/55 measured seconds, with 46/10/6 black startup frames. All three final
camera images were recorded as viewed, and native identity bracketed capture.

## Artifact and measurement boundaries

| Runtime stage | Bytes | SHA-256 |
| --- | ---: | --- |
| 134/135 in-place framing | 37,760 | `ccb38809c92f07b328e806e49f394e7d0b1e18a9bf650b9bb3d982353d6abd32` |
| 136/138 autonomous initialization | 44,208 | `ba9da35b0a2e2ce9a000eef54fd4da54da72d6883f8f210d6fde3d4c8171d6ca` |
| 139 continuous session, reported identity | 45,816 | `1d503812087e9da2c7f1ec57a3c62aa88668120da7e6f1b8c89da69f6b527807` |
| 141 loader JSON/OTP | 52,536 | `4232f1004dd39966b7bb8bdf5b81ebd8ec0ed491152f6c09858ed0a6eafef678` |
| 142 lifecycle | 53,624 | `97f32bb65362ab3c969c1e4e4e11011b05b141f3cbd8426677c580be595025ae` |

These early RAM containers retained template configuration; the later release
package no longer required a stock packaging template. A valid additive checksum
or manifest was not acceptance as an OTA/NAND image.

Captures used MACROSILICON `345f:2130`, serial `20210623`, at 1920×1080 YUYV and
nominal 30 fps. Their timestamps measure the capture adapter. They do not measure
RF/camera cadence or end-to-end latency; UI latency was especially unqualified
while AP-time synchronization was unsupported. Zero measured YUYV driver errors
are not zero HEVC decoding errors. Several captures had four error buffers in
the excluded warmup, while the measured window had none.

The harness restarted the original media stack by reloading unchanged stock
`ar_sys`/`ar_vb` with `mpp_service=lib`; simply restarting the backend had left
buffer allocation failures. It stopped the GUI supervisor before its child and
waited for teardown. A redundant cleanup helper did not solve those failures.
These details explain the experiment, not a universal hot-restart procedure.

Full recovery combined restoration, a new receiver boot, USB rediscovery,
24 getter checks and a separate viewed camera capture. After trial 132, the first
boot and settled retry still had six failing link queries and only background;
a second complete reboot restored camera video. A runner's exit status alone
was therefore insufficient. Later trials retained their own successful stock
recovery receipts, with no persistent replacement performed by this sequence.

Broader peer loss, modes/slots, fault ownership, calibration and command coverage
remained open at these stages. Later findings supersede some limitations, but
must be evaluated through their own [validation record](open8030-validation.md).
