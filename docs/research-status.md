# Research coverage and open questions

[Documentation index](../README.md)

This wiki begins with summaries of existing research. “Documented” below means
there is an article here; it does not mean every function or hardware variant is
fully understood.

| Area | Established within the recorded scope | Still open or bounded |
| --- | --- | --- |
| Ascent Gnd485 | AR9311 host, separate SDIO radio, Linux/media architecture and host HEVC extraction | Broader hardware revisions and complete peripheral documentation |
| Ascent Lite Sky482 | Firmware architecture, NAND identification and startup repair with visible stock video afterward | General recovery procedure across revisions and long-run post-repair reliability |
| AR8030 / Open8030 | Physical radio execution and later matched receiver/transmitter video bench | Full vendor parity, broader RF modes, hardware variants and recovery coverage |
| AR9311 media | H.264/H.265 full-frame decode and exact supplied regions after forced picture completion | Resumable slice decoding, reliable region-ready signal and early scan-out |
| AR9311 platform | GPIO readback, selected MMC clocks, watchdog expiry and fixed-mode HDMI tests | General clock/reset/power ownership and complete peripheral coverage |
| G11 | Host/MCU/radio architecture, custom MCU mixing and Linux video/UI integration | Physical CRSF delivery, wider model timing and remaining board services |
| BetaFPV-related pipeline | Recorded camera/media architecture and tiled HEVC transport behavior | Product/board identity mapping and compatibility across firmware families |

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

## Documentation priorities

1. Add board photos, component maps and measured connector/pinout information.
2. Expand AR9311 and AR8030 register, boot and peripheral studies.
3. Turn protocol overviews into field tables with named firmware scope and traces.
4. Add reproducible update/recovery and measurement records with artifact hashes.
5. Expand the BetaFPV and G11 application studies without transferring findings
   between boards solely because they share an Artosyn chip.
