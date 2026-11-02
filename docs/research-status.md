# Research coverage and open questions

[Documentation index](../README.md)

This wiki is undergoing a systematic review of the existing research. Coverage
is incomplete: some subjects have detailed evidence and field tables, while
others still have introductory summaries. “Documented” below means there is an
article here; it does not mean every source record, function or hardware variant
has been fully reviewed.

| Area | Established within the recorded scope | Still open or bounded |
| --- | --- | --- |
| Ascent Gnd485 | AR9311 host, separate SDIO radio, Linux/media architecture and host HEVC extraction | Broader hardware revisions and complete peripheral documentation |
| Gnd485 stock access | Root SSH through writable startup and RAM account overlays; one-shot external-script execution and reboot verification | Original password unrecovered; writable-etc console interaction prevents assuming binary-management coexistence |
| Gnd485 host ground station | Passive HEVC extraction, persistent TCP/RTP proxy, direct host playback and reboot recovery | Physical HDMI through the proxy, latency, runtime failover and tested keyframe trigger remain open |
| Gnd485 backend interfaces | Two read-only GUI getters and passive status-field decoding | Setter application/persistence, peer selection and full telemetry semantics remain unvalidated |
| Ascent PC/browser interfaces | Shared standard/Pro PC upload FSM, live receiver copy/readback, simulated Web Serial UI and bounded WebUSB/codec probes | Physical browser upload, custom bulk integration and browser HEVC playback unverified; cross-model target writers/keys remain distinct |
| Gnd485 Kestrel UI | Software HUD over observed live HDMI, validated MSP forwarding, CPU menu improvements and passive-worker recovery | Full UI/media controls, tearing/latency, resolution/reconnect behavior and reliable stock restoration remain incomplete |
| AR9311 stock GE2D | Exact offscreen RGBA copies, alpha behavior, RPC/MMZ layouts and measured job times | Transparent source-over, bounded DMA recovery, live UI integration and active-video stress unqualified |
| Gnd485 debug UART | Adapter rate clamping and scope evidence consistent with approximately 1.15 Mbaud, likely 1.8 V | Readable console and exact live frame format not established by the initial experiment |
| Gnd485 boot and updates | NAND/GPT and environment formats, verified USB RAM boot, stock updater flow and current fpvOS transaction boundaries | Shared kernel/DTB writes are not power-loss atomic; broader failure and bad-block hardware tests remain open |
| Ascent Lite Sky482 | Firmware architecture, repeatable NAND captures, reconstructed controller ECC/layout, restricted 37-block repair and visible stock video afterward | General recovery across revisions, BootROM loader/tool validation and long-run post-repair reliability |
| Ascent air/ground reuse | Matched host ABI/media differences, SKY/GND radio image/configuration comparison and Lite 17.5.8 updater trace | Stock-peer reception on repurposed Sky482, newer-product trust chains and complete update failure testing |
| AR8030 / Open8030 | Physical radio execution and later matched receiver/transmitter video bench | Full vendor parity, broader RF modes, hardware variants and recovery coverage |
| Early Open8030 AP trials | Offline role defaults, held Sky482 preparation, large host-write completion and bounded generated-data/video progress | Failed DMA/SDIO candidates and USB disappearance retained; short exact transfer is not sustained 35-Mbit/s video |
| AR9311 media | H.264/H.265 full-frame decode and exact supplied regions after forced picture completion | Resumable slice decoding, reliable region-ready signal and early scan-out |
| AR9311 platform | GPIO readback, selected MMC clocks, watchdog expiry and fixed-mode HDMI tests | General clock/reset/power ownership and complete peripheral coverage |
| G11 | Stock platform/connectors, autonomous MCU inputs/mixing, model activation and persistence, byte-identical decoded test pictures, CRSF contact and unloaded drive improvement | Connected CRSF faults/reset recovery, module replies, receiver delivery, wider model timing and remaining board services |
| P1 Air | SC231AI source capture/encode, AE/AWB, matched radio video, settings, calibration and direct NAND update/recovery | RF power/range, alternate modes, broader image-quality and failure validation |
| VR04-related reference pipeline | NT99235 sensor/CSI agreement, sustained fresh CVISP frames within successful sessions, ISP tables/statistics and tiled HEVC transport | Full camera restart/reset, raw bypass, adaptive tone/CLAHE and cross-board compatibility remain unqualified |
| VR04 board services | Reference panel/straps, shared SD rail, LED signaling, ADC/PWM/protemp source contracts and SDIO resynchronization | Board calibration, temperature accuracy, camera gaps and historical kernel flash qualification remain bounded |
| VR04 media reference | Exact tested decode, concurrent decoder/DVR workload, retained encoder-reuse failures and later reset/timeout/live-control/DMA changes | Inconsistent older rate-control claims, same-boot HEVC DVR reuse and general recovery qualification remain unresolved |
| VR04 RF/session reference | Media headers, vendor IDR/SEI requirements, identity/readiness, binding and later power/rate/standby implementation | Preset work-mode table, some MID getter meanings, normal-band scan layout, true association-loss signal and broad interoperability remain incomplete |
| DJI comparison | Goggles 3 static receiver graph, product profiles, host header and update-package metadata; historical O4 interface specifications | O4 payload decryption, actual air-side graph, RF schema, negotiated modes and measured latency remain unresolved |

## Corrections carried into these pages

- Early AR8030 diagnostic firmware did not implement RF/video. Later Open8030
  bench results demonstrate additional capability on a named firmware/board pair.
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
