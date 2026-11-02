# Open8030 validation and compatibility limits

[Experiments](README.md) · [AR8030](../soc/ar8030.md) · [Runtime](../firmware/open8030-runtime.md)

The strongest release result in this record is the 2026-10-22 Open8030 v0.2.0
bench with a regular Ascent VRX in DEV role and an Ascent Lite board-482 VTX in
AP role, both over SDIO with matched 18.21.10 inputs. It establishes working
radio/video operation for that pair. Full original-firmware coverage remains
unestablished.

The [early AP bring-up record](open8030-ap-bringup.md) separately covers held
preparation without RF start, sector-padding and large-write behavior, ownership
stalls, a short exact 33.86-Mbit/s generated-data transfer and a five-minute
12.27-Mbit/s video sample. None is an additional run of the release bench below.

## Matched AP/DEV release bench

| Observation | Recorded result |
| --- | --- |
| Duration | 299.71 seconds |
| Sampled link states | All 293 connected |
| Delivered bytes | 605,177,547 |
| Receiver queue drops | 7,988 bytes |
| AP write rejections | One queue-full and two out-of-order rejections |
| Recovery | Rejections recovered without intervention |
| Camera output | Visible scene captured through HDMI-to-USB |
| AP stop/start | Automatic recovery; admission list preserved |
| DEV stop/start | Passed on the preceding DEV image with the same DEV runtime behavior |

Release images were byte-identical to the final tested pair. The public loader
at that stage supported the VRX; the AP image required a board-specific RAM
loader. The bench separation was within one metre. Sampling did not exclude
subsecond picture freezes. An earlier fifteen-minute candidate soak recorded
no sampled stalls or queue drops, but preceded the AP restart fix; it is not
a fifteen-minute test of the released assets. This test does not establish
losslessness, range, interference tolerance,
35+ Mbit/s operation, USB VRX Pro support or full telemetry/RC parity.

The release increased the AP's pending-write capacity to five for the original
camera's pipelined traffic and retained its receiver admission list across
restart. The VRX loader used a unique firmware filename for each load to avoid
reusing a stale image. The AP build used `AR_ROLE_AP` and `AR_FACTORY_OPTIONAL`
with the matched original SDIO driver, configuration, daemon and camera services;
it was a radio image, not a complete Linux or flash-update package.

### Artifact identities

These identify the recorded DEV baseline and its inputs. They are not implied
hashes for the separate AP image.

| Artifact | Size / SHA-256 |
| --- | --- |
| DEV runtime | 115,332 bytes; `a1f97401c05b8764b205acb62289cbdd67b3f6f95a0c419b58b641908066f12d` |
| Bootstrap | 792 bytes; `c69e46c57bbb3e5fc7fe0c5581e59fb5735dcf8a97bf0e05ae7aaa0273657768` |
| Generated and accepted container | 133,632 bytes; `30e45c6aadb16049581f7de3f256781027eff4186b47f4a84814db6117d75c9a` |
| Board-profile input | `1bc673f086b67f2a742233619d655d92aca6816932a57e2b674cbdbd4f6f12c5` |
| External configuration | `5cf0cd31645d4786c6b04399c6c609386d427ffc4c6d7f6e42d81e3e770c2a74` |
| Matched stock radio container | `dc58247910af564376350404383c5c411cbec21663873fba24e70c61a3584e3c` |
| Stock main payload | `3e454131a334ec422c0b2e8ae4924d6bda68cb61b70fd754da26412578b39c87` |

## Earlier receiver milestones

These results belong to their historical images. They explain the reconstruction
and its remaining limits; they are not additional runs of the release above.
The [early DEV trial record](open8030-early-dev.md) details original-stack video,
source-bootstrap adoption, continuous operation across timer wrap, the stale
loader-JSON suffix failure and two explicit restart sequences.

The earlier v0.1.0 release was receiver-only, retaining the original transmitter
and receiver Linux software. Its 78,848-byte container had SHA-256
`fc8d2b94c336f0718f93d28811a36747cb04eeb8409acf99973df720af8fe552`.
That exact image had 55 seconds of visible HDMI-to-USB camera capture and
automatic retention in RAM verified. The prebuilt image included compiled
matched board data; reproducing it from source still required the private
profile. RAM retention did not establish persistent installation or AP support.

| Milestone | Evidence and conclusion | Boundary |
| --- | --- | --- |
| Source bootstrap and loader | Container built without a stock packaging template; normal loader JSON, identity and camera path exercised on 2026-10-14 | Board profile and matched proprietary host stack still required |
| AP time | 2,304 original-instruction cases; startup and two lifecycle sequences yielded 104 linked samples and visible video | DEV offset survives stop/start and resets on deinit/init; physical stale-peer and long-run wrap remain separate |
| Calibration lifetime | 128 original reset cases; hardware retains five compensation words through restart/reinit | Does not validate emitted power or all calibration banks |
| Bounded diagnostics | Ten complete RX records read in ranges, eleven invalid requests and end-boundary checks | Segmented live records are not atomic snapshots |
| Candidate recovery | Removal/readmission advances recovery counters without changing compensation words; camera returns | Not a physical peer-loss or interrupted-DMA stress test |
| Independent scan | Source sweep and score processing operate while link PLLs stay fixed | Stock observed four enabled measurement channels; native configured 42 and did not reproduce the host catalog workflow |
| TX MCS adaptation | Normal-key user-0 decrease observed as 2 → 1 → 2; 110-second run with 125 linked samples | Does not close other users, controlled loss or complete IRQ/ownership behavior |
| Cooperative close | 24 hardware close cycles, four active-write cancellations and zero drops in that window | Copy-mode lifetime differs from stock retained buffers; later streaming drops still occurred |
| Binding stages 233–234 | Physical button binding, saved peer, cancellation and reconnect after reboot/reload | Existing hardware pair; previously unknown identity only covered synthetically |

At the low-power investigation stage, changing only initial user-0 level from
25 to 11 in the accepted configuration failed acquisition. Restoring level 25
and rebooting restored the reference scene. This reproduced the failure without
the shelved dynamic-power implementation, but did not identify its RF cause or
measure transmitted power. Later P1 calibration work has a separate
[board-specific validation record](p1-source-stack.md).

The earlier [trials 143–161](ar8030-recovery-and-power.md) detail the first
candidate-removal timeout, two successful same-peer recovery cycles and native
factory-offset fallback. They also preserve stock acquisition at level 11,
native failure despite matching gain readbacks, and a compensation-ordering
correction that did not resolve acquisition. A level-25 override followed by
backend selection of 14 is not a sustained level-25 comparison.

The subsequent [trials 162–163](open8030-baseline-revalidation.md) physically
tested the final archived frame-duration build, which failed acquisition, then
revalidated camera video with the exact older stage-145 runtime. This supersedes
the handoff's earlier unused-trial statement without establishing a fix for the
newer implementation. Both milestones precede the AP/DEV release bench above.

The catalog mismatch also had a concrete host dependency: rejected getter
`0x0100000a` left the Linux backend's channel catalog empty, so it never sent
work-list setter `0x02000023`. An offline-tested catalog helper did not by itself
implement either command in the runtime.

## Drops, freshness and failed candidates

Short zero-drop windows did not establish a general streaming fix. Stage 234
recorded 299,118 dropped video socket bytes over 32 seconds after binding. A
separate sequential comparison recorded 199,212 bytes for stage 232 and 463,799
for stage 234, each over 29 seconds. Drops therefore predated the binding change;
these observations neither isolate their cause nor establish equal loss rates.

Flow instrumentation recorded both drop-bearing and zero-drop sessions. One
fresh 420-sample session had no recorded drops/errors, maximum queue occupancy
12,231 of 56,320 bytes and a maximum descriptor duration of 13.6 ms. Another
ten-minute, 600-sample session had zero drops. These are bounded observations,
not proof that instrumentation or a later helper eliminated loss.

Many earlier video checks showed a dark textured scene matching freshly booted
stock. That supports continuity of the tested path, without a controlled scene
freshness challenge or bright-scene quality comparison. Warm reloads sometimes
produced a black camera area or disconnected background despite a radio link;
full stock reboot restored the baseline in recorded sessions. Failed candidates
and warm-reload failures must not be relabeled as accepted baselines.

The HDMI-to-USB adapter is test equipment. Its nominal 30 fps output does not
measure the camera's rate, RF cadence or end-to-end latency. Live overlays can
continue while camera content is absent or stale.

## October 21 USB/AP integration checkpoint

An offline integration combined contributor USB changes `7321fa4` / `43c859a`
with AP development based on `e751313`. Public baseline `9f40a4b` and the
accepted release identities remained separate. This checkpoint did not load
either device and must not be read as a USB hardware acceptance result.

The central USB fix kept OUT-buffer ownership with the cooperative consumer
until its next receive call. Sending a reply while parsing could no longer
rearm DMA into those same bytes. AP transmission cleaned buffers before DMA;
completed AP reception invalidated them before parsing. Native tests checked
the handoff calls, without establishing physical cache coherency.

Four receive buffers grew to 48 KiB each. Oversized packets were rejected and
AP preparation/payload budgets capped before copying socket data. The combined
build retained 2-KiB cooperative host copies, 10-KiB input framing, four-block
SDIO output and the existing coding-reset ownership guards. These are candidate
implementation details, not measurements of increased throughput.

Several compatibility choices still required physical checks:

- Peer selection updated both the active and saved restart configuration,
  restoring persistence assertions removed by the contributor workaround.
  The contributor's original forced-USB run did not validate this revision.
- DEV startup retained its configured bandwidth instead of automatically
  requesting 20 MHz on a configured 40-MHz AP link. Outstanding RX DMA and TX
  during bandwidth, frame-mode and channel transitions remained review gates.
- Transport selection checked the loader's USB address, then SDIO configuration,
  otherwise choosing USB. Synthetic combinations and forced-transport tests
  passed; the automatic register heuristic still needed board validation.
- SDIO host power setters remained unsupported to preserve loader calibration.
  USB power handling had a separate validation scope. The socket status marker
  `0x12345678` also needed end-to-end checks with both host implementations.

The checkpoint changed project diagnostics; these are not vendor command IDs
or layouts guaranteed across later builds:

| Diagnostic | Candidate layout |
| --- | --- |
| Flash read | `0x0c5a0035`, avoiding AP prepare at `0x0c5a0030` |
| MCS information | Original 256-byte prefix plus four words; 272 bytes |
| PHY trace | Original 3,344-byte prefix plus RX/host-push timing; 3,368 bytes |
| Traffic information | Version 2, 800 bytes |

Exact-length consumers needed corresponding changes. A zero SDIO queue
high-water value meant unavailable reporting, not an observed empty queue.
Project register readback retained its held-state/configuration-page restriction;
active status reads were not presumed harmless.

The record reports 18 Python tests, 67 native suites and four QEMU UART cases
passing for the corrected contributor branch; the combined candidate passed
19 Python tests, 70 native suites and four QEMU cases. Its receiver-profile
link used 688,624 of 704,512 SRAM bytes, leaving 15,888 allocated bytes free.
That is neither measured stack margin nor AP-board acceptance. These historical
checks were not rerun during the documentation review.

Remaining hardware gates included fresh binding, restart, telemetry, visible
video and comparative throughput on SDIO, plus enumeration, transport selection,
sustained traffic and binding/restart on the contributor's USB configuration.
The later SDIO release bench above has its own image identities and scope.

## Contributor-reported AP results

Separate reports describe two air-unit boards with identical radio boot pin,
pad, I/O-domain and function-table setup. Their host-provided sub-band calibration
and front-end/power-offset calls differ. The AP build was reported byte-identical
across those boards.

Reported results include MCS 11 at 20 MHz and a 35.5 Mbit/s link. One board's
2026-10-20 report describes 20 Mbit/s video at 100 fps without decode errors,
a 4,650.9 µs short frame and roughly 21-second relink after VRX radio restart.
TX power changes were not covered. These are contributor-reported results with
a separate scope, not measurements from the five-minute release bench.

## What compatibility evidence establishes

The archived receiver replacement plan tracked a broader acceptance scope than
its successful video milestone. Its original trial-135 baseline and later
stage-162 additions should be read by checkpoint: older “current” RAM figures,
unused trial numbers and missing-feature lists were retained alongside newer
paragraphs. The final archived stage-162 build uses all 704,512 bytes of allocated
RAM, while the later AP/DEV release above is a separate tested implementation.

| Acceptance area | Historical evidence | What that evidence leaves open |
| --- | --- | --- |
| Executable provenance and initialization | Source bootstrap, matched loader JSON/OTP and original-stack video in trials 138/141 | Full configuration/board coverage and normal-load cold-start qualification |
| Host and socket lifecycle | Two explicit restart sequences in trial 142; later cooperative-close trials | Complete host commands, retained-buffer semantics, event producers and failure cases |
| Continuous operation | Three-minute trial 139 across low-counter wrap; later five-minute AP/DEV bench | Long-duration resource bounds, interference and sustained faults |
| Recovery and pairing | Same-peer candidate cycles in trial 144; later binding/button/reboot tests | Physical peer-loss/restart matrix, previously unknown peers and interrupted-DMA recovery |
| RF configuration and synchronization | Matched fixed settings, MCS, then archived offline channel/sync/automatic-scan integration | Broad mode/board behavior, physical automatic policy, emitted power and RF timing |
| Platform and deployment | Experimental polling port and bounded RAM loads with stock restoration | Complete interrupt/timer port, memory margin, final normal-load package and rollback validation |

Source-built executable payloads still need separately identified board inputs.
Similarly, successful RAM packaging or stock restoration does not establish a
persistent installation or native cold boot. These are historical acceptance
boundaries, not instructions to change the device during documentation work.

### Evidence levels and command coverage

Four levels need separate tracking:

1. **Recovered stock contract:** dispatch, fields, errors, preconditions, timing
   and side effects tied to a particular original payload.
2. **Offline comparison:** independently implemented behavior compared against
   original instructions, with intercepted helpers and synthetic peripherals
   declared. Native tests alone do not constitute this comparison.
3. **Runtime integration:** normal loader/backend paths execute the behavior,
   including outstanding work and notifications.
4. **Matched hardware comparison:** stock and replacement run the same scenario
   with exact image/configuration identities and inspected camera output.

The core command inventory found 114 distinct IDs across dispatch and reply-size
tables, of which 108 had recognized routes. The reply-size table had 99 entries,
93 shared with the dispatcher. Routing and size comparisons covered 4,811 and
1,537 cases respectively. These counts are not implemented features or a
percentage of firmware compatibility: handler bodies, allocation, serialization,
callbacks and hardware effects are separate.

Six IDs had reply-size entries but fell through core default rejection:
`0x0100006e`, `0x0100006f`, `0x02000017`, `0x02000019`, `0x0200001a` and
`0x0a000004`. Other layers may handle them. Conversely, some initialization and
setter routes lacked reply-size entries. Core recognition does not prove normal
host transport availability.

The original core default returns `-4`; native unsupported operations commonly
return `-95`. Wire-layer mappings need separate comparison. A 64-entry request
audit records observations, not response success or caller identity, and can
overflow. Diagnostic IDs are not vendor compatibility coverage.

Open areas include complete host handlers, event producers, UART/SBUS services,
retained-buffer ownership, RTOS scheduling, automatic channel policy, wider
power/mode/board coverage, sustained faults and complete telemetry/control
effects. Later AP and P1 successes supersede early blanket statements that only
receiver operation or zero-offset factory fallback had ever been demonstrated;
they do not close every item in the historical receiver audit.
