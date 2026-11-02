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
| Gnd485 boot and updates | NAND/GPT and environment formats, verified USB RAM boot, stock updater flow and current fpvOS transaction boundaries | Shared kernel/DTB writes are not power-loss atomic; broader failure and bad-block hardware tests remain open |
| Ascent Lite Sky482 | Firmware architecture, NAND identification and startup repair with visible stock video afterward | General recovery procedure across revisions and long-run post-repair reliability |
| AR8030 / Open8030 | Physical radio execution and later matched receiver/transmitter video bench | Full vendor parity, broader RF modes, hardware variants and recovery coverage |
| AR9311 media | H.264/H.265 full-frame decode and exact supplied regions after forced picture completion | Resumable slice decoding, reliable region-ready signal and early scan-out |
| AR9311 platform | GPIO readback, selected MMC clocks, watchdog expiry and fixed-mode HDMI tests | General clock/reset/power ownership and complete peripheral coverage |
| G11 | Host/MCU/radio architecture, autonomous MCU mixing, CRSF output contact and unloaded drive improvement | Connected CRSF faults/reset recovery, module replies, receiver delivery, wider model timing and remaining board services |
| P1 Air | SC231AI source capture/encode, AE/AWB, matched radio video, settings, calibration and direct NAND update/recovery | RF power/range, alternate modes, broader image-quality and failure validation |
| VR04-related reference pipeline | NT99235 camera/media architecture and tiled HEVC transport | Separate board identity and cross-firmware compatibility |

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

1. Add board photos, component maps and measured connector/pinout information.
2. Expand AR9311 and AR8030 register, boot and peripheral studies.
3. Turn protocol overviews into field tables with named firmware scope and traces.
4. Add reproducible update/recovery and measurement records with artifact hashes.
5. Expand the BetaFPV and G11 application studies without transferring findings
   between boards solely because they share an Artosyn chip.
