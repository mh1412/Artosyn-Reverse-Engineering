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
loader. This test does not establish losslessness, range, interference tolerance,
35+ Mbit/s operation, USB VRX Pro support or full telemetry/RC parity.

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
