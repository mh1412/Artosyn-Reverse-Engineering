# Research coverage and open questions

[Documentation index](../README.md)

This wiki is undergoing a systematic review of the existing research. Coverage
is incomplete: some subjects have detailed evidence and field tables, while
others still have introductory summaries. “Documented” below means there is an
article here; it does not mean every source record, function or hardware variant
has been fully reviewed.

The inventoried Markdown/RST research, including hidden project notes and
retained checkpoints, has been reconciled or explicitly classified by relevance.
Copied SDK, kernel, tool and build documentation is classified separately and
does not count as original research reviewed. The broader work remains open:
raw captures, aggregate experiment records, implementation files and visual
evidence have differing levels of review. Closing the narrative inventory is
not a claim of complete reverse engineering or complete source-code coverage.

A fresh discovery pass across the requested projects also included ignored files
and nested hidden research folders. It found no additional original Markdown/RST
narratives to incorporate; newly discovered tool and dependency documentation was
classified separately. This check does not close the review of prose logs, JSON
results, captures, images or implementation files.

| Area | Established within the recorded scope | Still open or bounded |
| --- | --- | --- |
| Ascent Gnd485 | AR9311 host, separate SDIO radio, Linux/media architecture and host HEVC extraction | Broader hardware revisions and complete peripheral documentation |
| Gnd485 stock access | Root SSH through writable startup and RAM account overlays; one-shot external-script execution and reboot verification | Original password unrecovered; writable-etc console interaction prevents assuming binary-management coexistence |
| Gnd485 host ground station | Passive HEVC extraction, persistent TCP/RTP proxy, direct host playback and reboot recovery | Physical HDMI through the proxy, latency, runtime failover and tested keyframe trigger remain open |
| Gnd485 backend interfaces | Two read-only GUI getters and passive status-field decoding | Setter application/persistence, peer selection and full telemetry semantics remain unvalidated |
| Ascent PC/browser interfaces | Shared standard/Pro PC upload FSM, live copy/readback, later physical hosted-browser fpvOS install/self-update/cold boot/stock restore, bounded WebUSB/codec probes | Exact browser release tested on one Gnd485/stock-bank pair; custom bulk integration and browser HEVC playback unverified; cross-model target writers/keys remain distinct |
| Gnd485 Kestrel UI | Software HUD over observed live HDMI, validated MSP forwarding, CPU menu improvements and passive-worker recovery | Full UI/media controls, tearing/latency, resolution/reconnect behavior and reliable stock restoration remain incomplete |
| AR9311 stock GE2D | Exact offscreen RGBA copies, alpha behavior, RPC/MMZ layouts and measured job times | Transparent source-over, bounded DMA recovery, live UI integration and active-video stress unqualified |
| Gnd485 debug UART | Adapter rate clamping and scope evidence consistent with approximately 1.15 Mbaud, likely 1.8 V | Readable console and exact live frame format not established by the initial experiment |
| Gnd485 boot and updates | NAND/GPT and environment formats, verified USB RAM boot, stock updater flow and current fpvOS transaction boundaries | Shared kernel/DTB writes are not power-loss atomic; broader failure and bad-block hardware tests remain open |
| VRX Pro fpvOS | Source-reviewed SD boot edits, kernel/graphics packaging, board-service policy and pairing merge; beta4 application and optical timing report | No new Pro hardware test; UBI attachment prevents a blanket no-NAND-writes guarantee; reported latency is specific to Lite+/panel/firmware and raw samples were not reprocessed |
| Ascent Lite Sky482 | Firmware architecture, repeatable NAND captures, reconstructed controller ECC/layout, restricted 37-block repair and visible stock video afterward | General recovery across revisions, BootROM loader/tool validation and long-run post-repair reliability |
| Ascent air/ground reuse | Matched host ABI/media differences, SKY/GND radio image/configuration comparison and Lite 17.5.8 updater trace | Stock-peer reception on repurposed Sky482, newer-product trust chains and complete update failure testing |
| AR8030 / Open8030 | Physical radio execution and later matched receiver/transmitter video bench | Full vendor parity, broader RF modes, hardware variants and recovery coverage |
| Early AR8030 physical bring-up | SDIO ordering fix, bounded queued echoes, native calibration and separate stock clock/GPIO observations | Peripheral completion and matching static settings did not establish reception; restored identity did not guarantee video recovery |
| Early AR8030 acquisition | Search/profile reconstruction, isolated-stock candidate dependency and independently checked native MAC control packets | Hardware validity was insufficient; FCH diagnostics, admission, linked operation and video were separate milestones |
| Early AR8030 uplink/login | Copied TX ownership, stock report comparison, matching AP assignment, compact controls and bounded recovery | Frequent recovery cycles, truncated payload captures and local TX completion did not establish full PHY policy or video |
| Early AR8030 host/video | Host socket delivery, remote application ACK, negotiated PHY identity/MCS, session sync and strictly decoded retained suffixes | Private probes stopped the original backend; retained prefixes and local endpoint counts do not establish whole-stream/RF integrity |
| Early Open8030 DEV integration | Original-stack video, source bootstrap, timer-wrap crossing, corrected loader JSON/OTP and two host restart sequences | Bounded RAM observations; stuck-DMA paths synthetic, capture-card cadence not RF timing, later release behavior separate |
| Early AR8030 recovery/power | Two same-peer candidate cycles, factory-offset fallback, gain/state comparisons and compensation ordering | Pending-DMA shutdown untested physically; stock acquired at level 11 while native failed; matching calculations/readbacks did not establish RF policy |
| Archived AR8030 channel integration | Offline subchannels, directional tuning, hop monitor/rollback, remote sync, automatic-scan lifecycle and corrected physical-frame duration; later trial 162 failed acquisition and trial 163 restored video with stage 145 | Newer implementation's regression unresolved; physical automatic-policy/timing acceptance unproven; final development build has no allocated RAM headroom |
| Early Open8030 AP trials | Role defaults, held preparation, C/D pin-mux correction, native acquisition, exact short bidirectional streams, first camera video and later throughput trials | Repeated startup failures, DMA/SDIO stalls and USB disappearance retained; short exact transfer is not sustained 35-Mbit/s video |
| AR9311 media | H.264/H.265 full-frame decode and exact supplied regions after forced picture completion | Resumable slice decoding, reliable region-ready signal and early scan-out |
| AR9311 platform | GPIO readback, selected MMC clocks, watchdog expiry and fixed-mode HDMI tests | General clock/reset/power ownership and complete peripheral coverage |
| G11 | Stock platform/connectors, autonomous MCU inputs/mixing, model activation and persistence, byte-identical decoded test pictures, CRSF contact and unloaded drive improvement | Connected CRSF faults/reset recovery, module replies, receiver delivery, wider model timing and remaining board services |
| G11 vendor SDK | Complete manual reconciliation, stock video IPC layouts, offline activity/blocking probes and reproduced radio-wrapper failure masking | Selected implementation review, not full SDK audit; gimbal APIs, general update/recovery, hardware brightness limits and stock/custom binary equivalence remain bounded |
| Nomad / ELRS comparison | Physical USB RC through a receiver to FC readback, returned telemetry, stream-loss/recovery and TX profile persistence | Two-receiver switching failed despite verified TX settings; GPS fixture-only; no G11 module-bay or Artosyn RF acceptance |
| P1 Air | SC231AI source capture/encode, AE/AWB, matched radio video, settings, calibration and direct NAND update/recovery | RF power/range, alternate modes, broader image-quality and failure validation |
| VR04-related reference pipeline | NT99235 sensor/CSI agreement, sustained fresh CVISP frames within successful sessions, ISP tables/statistics and tiled HEVC transport | Full camera restart/reset, raw bypass, adaptive tone/CLAHE and cross-board compatibility remain unqualified |
| VR04 board services | Reference panel/straps, shared SD rail, LED signaling, ADC/PWM/protemp source contracts and SDIO resynchronization | Board calibration, temperature accuracy, camera gaps and historical kernel flash qualification remain bounded |
| VR04 kernel ABI program | Matched-build panic diagnosis, selected MMZ/PTS/scaler results and source-reviewed diagnostic checks | Full MPP conformance program parked; several modules only load-tested; scaler tolerance is not exact filter parity; SD backup/timeout behavior does not guarantee restoration |
| VR04 media reference | Exact tested decode, concurrent decoder/DVR workload, retained encoder-reuse failures and later reset/timeout/live-control/DMA changes | Inconsistent older rate-control claims, same-boot HEVC DVR reuse and general recovery qualification remain unresolved |
| VR04 stock MPP | Manually pumped decode/Fusion/client graph, in-process service ownership and recovered codec command/attribute layouts | Conflicting VDEC/VENC frame offsets, exact pixel-format mapping and independent stock-side codec attachment remain unvalidated |
| VR04 encoder and latency experiments | Later synthetic detail fits the datagram budget; separate low-load picture test; receiver first-packet-to-latch improvement | Noise still exceeds packet budget; full camera-to-screen timing, pixel-clock calibration and overflow recovery test remain open |
| VR04 HUD and canvas | Open composite/overlay ownership, optional FC-OSD recording burn-in, 68 captured canvases reconstructed byte-for-byte | MSPv1-only open client; nonzero attributes, marker/separator content, empty-canvas behavior and broader playback/restart qualification remain bounded |
| VR04 RF/session reference | Media headers, vendor IDR/SEI requirements, identity/readiness, binding and later power/rate/standby implementation | Preset work-mode table, some MID getter meanings, normal-band scan layout, true association-loss signal and broad interoperability remain incomplete |
| DJI comparison | Goggles 3 static receiver graph, product profiles, host header, latency mechanisms and update-package metadata; historical O4 specifications; Ascent exporter boundary probe | O4 payload decryption, actual air-side graph, RF schema, feature activation and matched optical latency remain unresolved |

## Corrections carried into these pages

- The [complete stock decoder-command review](soc/ar9311-decoder-commands.md)
  matches 1,382 instructions and reconstructs all 98 jump-table entries. Related
  SDK command ordinals differ; queue status is cached, display-backend failures
  can be hidden, and clearing allocation bookkeeping does not free DMA memory.
  The matching C ABI, called backends and broader library remain incompletely
  reviewed; this adds no hardware or partial-picture-output result.
- The [complete low-delay-helper and Wave5 submission review](experiments/ar9311-slice-decoding.md#complete-low-delay-helper-and-wave5-submission-review)
  matches 756 instructions to the retained library, corrects the reversed option
  shifts, and distinguishes metadata assembly and command acceptance from usable
  decoded rows. Matching duplicate binaries adds no independent hardware result;
  the broader library review remains incomplete.
- The [VR04 device-tree review](firmware/vr04-device-tree.md) accounts for the
  ramoops reservation below the shared codec/userspace media pool, records the
  complete declared flash map, and distinguishes legacy metadata from deployed
  hardware drivers. Offline compilation and property checks passed with three
  retained warnings; they do not establish runtime ownership or memory retention.
- The [VR04 benchmark-runner review](experiments/vr04-air-encoder-benchmark.md#current-runner-and-result-interpretation)
  identifies a shared stall timer that can hide a failed tile, cumulative fields
  in per-tier summaries, and a zero exit status after logged failures. It also
  separates GStreamer staggered startup from direct-camera open order and
  qualifies the noise generator's entropy/range. Offline checks reproduce the
  selected runner and renderer behaviors without new performance measurements.
- The [VR04 air capture/encoder review](systems/vr04-air-capture.md) separates
  buffer-credit release from verified DMA quiescence and live controls from
  transactions. Offline probes reproduce partial updates, an uncancelled startup
  timeout, and stale running state with zero-reference capture credit after
  every tile retires. These are implementation limits, not new hardware tests.
- The [VR04 air timing review](experiments/vr04-air-encoder-benchmark.md#what-the-air-latency-instrumentation-measures)
  distinguishes local send completion from RF delivery and shows that the
  no-transmit mode also advances `tx`. Trace attribution uses a FIFO; air p99
  uses a different rank rule from receiver p99. Offline accumulator checks
  reproduce negative-duration sentinels and skew pairs spanning reporting
  windows. These findings add no camera-to-screen or hardware measurement.
- The [VR04 pacing/diagnostic review](systems/vr04-display-diagnostics.md)
  distinguishes the current adaptive ready-to-event controller from the older
  submit-to-event measurements. Clock-write success does not prove actuator
  lock, and playback retains some controller history. Forced-phase counters
  and persistent markers have narrower meanings than their names suggest.
  Counter rendering was checked offline; its camera-feedback subtraction is
  not by itself a validated optical camera-to-screen measurement.
- The [VR04 instrumentation review](experiments/vr04-receiver-latency.md#what-the-current-instrumentation-counts)
  separates main latency, two-tile and cadence populations. Its receive mark
  follows validation/copying; a long flip gap counts one repeat, and a missing
  population reports zero rather than a measured zero latency. The
  [DRM cleanup review](systems/vr04-frame-ownership.md#broker-lifetime-splash-adoption-and-orphan-cleanup)
  also shows that reclaimed totals can advance despite failed removals and can
  double-count shared objects. Offline probes confirm these local behaviors;
  historical captures and physical reclamation were not retested.
- The [VR04 compositor review](systems/vr04-compositor.md) corrects stale heap
  defaults and allocation-size comments. A failed post-blend clean leaves the
  CPU-modified band in a composite that the caller still submits; an offline
  mocked probe reproduces that control flow. The NEON arithmetic test compiled
  but could not run without QEMU, and its optional test target can report a
  skip. Neither arithmetic equivalence nor a seam-skip counter proves DMA/cache
  correctness or a visible result.
- The [VR04 recording/restream review](systems/vr04-recording-restream.md)
  separates encoder-start state from encoded output and successful MP4
  finalization. Offline checks reproduce overlapping SRT cues, incomplete
  parameter-prefix suppression and an RTSP keyframe gate left open after failed
  pushes. The actual control dispatcher also truncates a maximum-size OSD cell
  by one byte; its receiver does not enforce the sender's shorter text limit.
  These checks add no storage, client-decoding or hardware acceptance.
- The [VR04 frame-ownership review](systems/vr04-frame-ownership.md) identifies
  a lost-flip timeout call rejected by its own token check, a queue threshold
  that can overshoot, and unsigned wrap in the datagram-length guard. Offline
  extracted-function checks reproduce those branches without hardware or network
  traffic. Current tile-layout fallback also places chroma after 552 image rows;
  allocation alignment alone must not move it to row 560. The retained rebuild
  helper is unused, and attempted display parking does not prove quiescence.
- The [VR04 playback review](systems/vr04-playback.md) separates the HUD's
  metadata-derived clip length from a pipeline duration that can still grow.
  Its recording-list size filter does not prove decodability, and scaling
  requires an existing composite pool. EOS replay rebuilds the graph; direct
  resume/speed commands leave ended state and its stopped timer intact.
  Published state and attempted live recovery do not prove successful decoder
  transitions or visible output. These are source findings, not new bench tests.
- The [VR04 session-handler review](protocols/vr04-video-session.md#what-the-open-handlers-actually-acknowledge)
  distinguishes expected packet formats from permissive receive checks and
  local send attempts. Failed configuration sends can still advance state.
  The [channel-survey review](protocols/vr04-channel-survey.md) also shows that
  a later sample need not match the initially gated channel, while cached tune
  state and survey completion do not prove physical restoration. Offline
  handler probes reproduce these limits without new radio tests.
- The [Kestrel HiSilicon encoder review](systems/kestrel-air.md#hisilicon-encoder-control-and-read-boundaries)
  distinguishes requested configuration from accepted SDK state. Removing a QP
  override does not necessarily restore its default, failed updates can be
  cached, and an oversized slice can return success with zero payload. Offline
  mocked checks reproduce these wrapper behaviors without adding hardware or
  vendor-ABI acceptance.
- The [AR8030 recovery capture review](experiments/ar8030-recovery-and-power.md#retained-capture-evidence)
  reconciles 25 capture reports and 50 first/last raw-frame identities. Retained
  images show why nonblack HDMI frames can still mean disconnected receiver UI.
  Missing or changed historical source references limit full checkpoint
  reconstruction; capture hashes and UI overlay values do not establish RF
  timing, firmware parity or continuous fresh camera motion.
- The [ISP provenance audit review](soc/ar9311-nt99235-isp.md#what-the-provenance-count-covers)
  reproduces the 1,260-offset model but corrects its interpretation as a complete
  executed-write or arithmetic audit. Offset-based explanations and the global
  skip list leave specific blind spots. Separately,
  [compiled coefficient packers](soc/ar9311-isp-coefficient-formats.md#offline-verification-and-its-boundary)
  match the pinned library's gamma companion, static DRC banks and compander
  byte-for-byte; that does not validate dynamic tone selection or DMA lifetime.
- The [NT99235 ISP runtime review](soc/ar9311-nt99235-isp-runtime.md) separates
  successful startup from complete memory ownership and working interrupts.
  It documents statistics-record consistency limits, in-place tone updates,
  the LTM upload's high-water-mark acceptance and incomplete diagnostic coverage.
  The selected top-level register check passed offline; a synthetic fixture
  exposed an unchecked interior span in the older default-copy verifier.
- The [NT99235 CVISP source review](soc/ar9311-nt99235-camera.md#capture-metadata-and-diagnostic-controls)
  identifies completion-time timestamps, queue-starvation behavior and the lack
  of an explicit DMA drain on stream-off. Its
  [table reconstruction](soc/ar9311-nt99235-isp.md#cvisp-table-provenance-and-verification-limits)
  reproduces two headers from the pinned library, but the derivation checker
  passes without live captures and retains eight recorded register values.
  BLC fallback constants and a mislabeled stride also qualify the checker's
  claim that all carried values are replaced before use.
- The [VR04 media patch review](experiments/vr04-media-validation.md) separates
  software job completion from verified DMA quiescence. It also corrects the
  zero-valued drain-timeout description: the code keeps one 60-second deadline,
  while zero disables the separate encode-job timeout. Overflow ownership,
  deferred keyframe changes and broad hardware-quirk conditions retain their
  specific implementation and validation limits.
- The [VR04 kernel ABI review](experiments/vr04-kernel-abi.md) qualifies the
  archived “all modules passed” statement with its unchecked conformance
  criteria. Later scaler tests check pixels, superseding completion-only notes,
  but retain filter, synchronization and input-dimension limits. SD diagnostics
  also retain a volatile default backup and cannot terminate blocked kernel I/O.
- The [VRX Pro source review](firmware/vrx-pro-fpvos-boot.md) qualifies the
  SD-only installation description: stock pairing data is mounted read-only,
  but UBI attachment can perform maintenance. Its source-built kernel and
  Panfrost path also supersede older proprietary-kernel/GPU descriptions.
  [Optical latency results](experiments/vrx-pro-latency.md) remain distinct from
  internal capture-to-display statistics and from AR9311 partial-picture tests.
- The [AP acquisition history](experiments/open8030-ap-acquisition.md) fills the
  gap between digital TX activity and native camera video. Eight C/D pin
  assignments preceded the first link; exact short streams and first camera
  startup were later checks. [Throughput experiments](experiments/open8030-ap-bringup.md)
  correct a one-acknowledgement-per-TCP-write assumption and separate offered
  rates, exact receipt, camera delivery and persistent DMA failures. Passing
  video samples did not exercise the coding-reset deferral overlap on hardware.
- The earlier G11 Linux-mixer runtime had separate
  [35.8-minute timestamp-admission](firmware/g11-uart-and-session-lifecycle.md#long-session-timestamp-defect)
  and [completed-ADC-scan timeout](soc/g11-gd32-acquisition.md#raw-inputs-and-dma-publication)
  defects. Their targeted fixes and short hardware observations do not turn the
  interrupted 45-minute or failed 75-minute monitors into endurance passes.
- The [G11 SDK review](systems/g11-vendor-sdk.md) resolves apparent Wi-Fi
  capacity conflicts as separate UI/HAL limits and corrects flattened RC
  structure tables. Its radio wrapper can return zero and delete the image
  after updater failure. The [video-control study](protocols/g11-stock-video-control.md)
  also shows that activity can become true without new input and that a receive
  timeout does not bound a partially received message. These are pinned source
  and offline results, not new hardware acceptance.
- [Nomad control-link records](systems/nomad-usb-control.md) supersede early
  settings-only scope with physical RC and FC telemetry. The separate
  [profile protocol](protocols/nomad-usb-profile.md) distinguishes accepted TX
  identity from actual receiver response: both two-receiver attempts stopped
  at A. A temporary-file rename also does not establish atomic persistence of
  the complete identity and radio configuration across power loss.
- The [hosted-browser hardware record](experiments/gnd485-browser-update.md)
  supersedes the October 31 fpvOS publication note that its exact release had
  not yet been hardware flashed. Native Web Serial completed install,
  self-update, cold-boot verification and stock restoration. Zero platform-check
  failures do not add RF, decoded-video or HDMI validation for that build.
- The [video-latency comparison](systems/video-latency-comparison.md) verifies
  that Ascent's network exporter can wait beyond the next first-slice header
  before closing the previous picture. This is a source-level dependency, not
  a measured millisecond penalty or a stock-HDMI finding. DJI timing thresholds,
  slice symbols and buffer counts likewise do not establish active O3 settings
  or a numerical glass-to-glass advantage.
- The [AR8030 analysis-method review](references/ar8030-analysis-method.md)
  distinguishes original image bytes from synthetic ELF flags, permissions,
  symbols and ABI choices. Generic decompiler truncation around custom
  instructions is not proof of unreachable code, and a valid identity-bound
  receipt can still contain an incomplete trace. Historical acceptance tables
  are interpreted by checkpoint rather than treating every retained “current”
  statement as the latest result.
- Early AR8030 diagnostic firmware did not implement RF/video. Later Open8030
  bench results demonstrate additional capability on a named firmware/board pair.
- The [physical bring-up history](experiments/ar8030-sdio-and-rf-bringup.md)
  records how a longer CIS masked the SDIO notification race. Correcting the
  host-mask handshake established reliable diagnostic exchanges; later PLL
  address corrections still left receive headers invalid. Stock observer runs
  and HDMI grabber cadence remain separate from native RF/video acceptance.
- The [acquisition experiments](experiments/ar8030-receive-acquisition.md)
  separate changing power, header validity and checked DMA contents. The first
  packets arrived after about seven seconds; earlier 1.5-second trials could not
  rule that out. Failed FCH register checks also coexisted with valid MAC controls.
  A successful padding comparison supports a measured loader workaround while
  leaving the exact ROM constraint unresolved.
- The [uplink/login history](experiments/ar8030-uplink-and-login.md) distinguishes
  three local TX completions from later AP acceptance. Corrected TX profile and
  startup gains preceded login; request-ID recovery then extended TX to the
  bounded deadline. A stock observer's report RAM establishes its sampled state,
  while its incorrectly selected GET_STATUS prefix does not.
- The [first-video history](experiments/ar8030-first-video.md) separates a
  keyframe-command ACK from later parameter sets and decoding. FCH identity
  synchronization removed a receive-policy failure, while session commands and
  copy-service changes were still needed. The matched host-layout study also
  corrects early status-offset guesses, write-completion payloads and buffer size.
- The [early DEV trial history](experiments/open8030-early-dev.md) preserves the
  failed bootstrap omission and loader-overlay attempt before their successful
  follow-ups. A packaging manifest's unverified-boot flag predates the hardware
  receipt; continuous firmware still had a bounded external observation window.
- The [recovery/power history](experiments/ar8030-recovery-and-power.md) retains
  a shutdown timeout before two candidate-driven recoveries. Neither successful
  shutdown had pending RX DMA. Factory zero-offset fallback was not a successful
  factory-record read, and a compensation-ordering correction did not fix native
  level-11 acquisition. Stock's successful level-11 comparison bounds that failure.
- The [channel checkpoint review](protocols/ar8030-channel-integration.md)
  distinguishes an experimental integration from the later parity audit's smaller
  runtime scope. Original instructions correct a mislabeled hop deadline to
  missed-count plus 16, rather than loss-threshold plus 16. Two later calibration
  corrections also failed to resolve physical level-11 acquisition.
- The [automatic-scan review](protocols/ar8030-automatic-channel-scan.md) separates
  independent power-survey, PHY and automatic-search histories. Readiness does
  not guarantee fresh measurements for every channel, capability does not enable
  policy, and cold loader grouping remains disabled until explicitly activated.
  The later [frame-duration checkpoint](firmware/ar8030-command-engine.md) fixes
  the old three-command sum by reading the installed physical slots, including
  preparations and waits. Its offline success does not establish RF timing.
  [Later hardware trials](experiments/open8030-baseline-revalidation.md) supersede
  the archived “trial 162 unused” statement: that build failed acquisition,
  while trial 163 restored video by reusing stage 145. This was baseline
  revalidation, not a fix to the newer implementation.
- Ascent Lite recovery progressed from a proposed restoration workflow to a
  direct-flash repair with normal boot and visible stock video verified.
- AR9311 GPIO direction registers were read successfully; the earlier blanket
  unreadability assumption is not supported by the later measurements and vendor
  driver analysis.
- Finalizing an incomplete picture can reconstruct the supplied slices, but does
  not establish that the same picture can remain open and resume decoding later.
- Stock-kernel and custom-kernel fpvOS updaters have different write/recovery
  boundaries and must be evaluated separately.
- P1 source work progressed beyond RAM-only boot to a direct NAND implementation
  with preserved stock bank and independent clean recovery. Its SC231AI camera
  must be distinguished from the NT99235 reference platform.
- NT99235 frame-start counters and plausible retained images did not establish
  fresh DMA output. The later capture driver keeps the upstream chain running;
  queue reopening is not a proven full camera reset. The
  [camera validation](experiments/nt99235-camera-validation.md) preserves the
  marker tests, restart failure and later workaround. ISP tone selection also
  has a separate scalar from the gain-driven noise stages.
- The later reference radio daemon emits 104-byte media replies, resets session
  timestamps on identity hello, and keeps media parameter polling active. Its
  air standby no longer waits for an ACK; the two-minute silence policy releases
  commanded power while retaining standby. The
  [link-control study](systems/vr04-link-control.md) corrects older notes and
  distinguishes these policies from physical RF qualification.
- The [air encoder benchmark](experiments/vr04-air-encoder-benchmark.md) contains
  results from different settings: later CU-level rate control made detail fit
  one datagram, while noise still exceeded it. VBV was not a hard picture-size
  cap, and GOP 65535 was outside the accepted control range. A later low-load
  picture test supersedes the old blanket absence of visual verification.
- The [receiver latency study](experiments/vr04-receiver-latency.md) reports
  25.1 ms median from first packet to display latch after two combined changes.
  It excludes upstream camera/encoding and panel optical response. Two retained
  pixel-clock mappings disagree, so the early 62.08 Hz refresh calculation is
  not a settled calibration.
- The [open media/HUD study](systems/vr04-open-media.md) replaces early
  blocking-display-only and automatic-return-at-EOS descriptions with the later
  flip-event path and pinned final-frame behavior. Buffer retirement accounts
  for possible event/latch skew, limiting interpretation of software latency
  timestamps. [Canvas checks](protocols/vr04-msp-canvas.md) pass the retained
  capture but expose synthetic marker/separator cases outside that corpus.
- Early MissingLynk/fpvOS comparisons predate the AR9311 fpvOS port. The
  [layer comparison](systems/open-receiver-stacks.md) links later board-specific
  results instead of preserving “RK3568 only” as a current limitation. The
  proposed LVGL frontend also progressed to verified stock-kernel UI/video
  coexistence; its later GE2D prototype remains separate from live rendering.
- Open8030's later runtime enables instruction/data caches while continuing to
  poll with CPU interrupts masked. Earlier build sizes and receiver-only scope
  do not describe the later AP/DEV release. The
  [validation record](experiments/open8030-validation.md) separates these stages,
  contributor reports and remaining compatibility gaps.

- G11 CRSF output was later located on RX/PPM; the earlier static-high capture
  is not the final electrical result. Improved unloaded drive still did not
  establish a working connected Nano link. The
  [dated experiments](experiments/g11-crsf-electrical.md) preserve that distinction.

## Documentation priorities

Review includes hidden research directories and nested radio/transmitter projects,
not only public Markdown pages. Deployment reports, failed attempts and later
corrections can supersede a project's introductory README. Duplicate build trees,
copied release snapshots and generated files must be distinguished from independent
evidence. Discovery or indexing alone does not mean a record has been read and
incorporated. The source corpus has not yet been fully reconciled.

1. Add board photos, component maps and measured connector/pinout information.
2. Expand AR9311 and AR8030 register, boot and peripheral studies.
3. Turn protocol overviews into field tables with named firmware scope and traces.
4. Add reproducible update/recovery and measurement records with artifact hashes.
5. Expand the BetaFPV and G11 application studies without transferring findings
   between boards solely because they share an Artosyn chip.
