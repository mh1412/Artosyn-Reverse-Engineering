# Research coverage and open questions

[Documentation index](../README.md)

The documentation now focuses on stock hardware and firmware. It retains binary
reconstruction, protocol captures, device observations and diagnostic experiments.
Documented findings do not imply that every source file, firmware function,
product revision or physical failure path has been reviewed.

## Coverage

| Area | Documented findings | Still open or bounded |
| --- | --- | --- |
| Ascent Gnd485 | AR9311 host, Linux services, USB management, NAND/GPT, root access and compressed-video extraction | Broader board revisions, complete peripheral wiring and stock failure recovery |
| Ascent Lite Sky482 | Host/radio architecture, stock ASW updater, raw NAND layout and restricted flash repair | General BootROM recovery and behavior across firmware revisions |
| AR8030 | Matched image identities, loading, RISC-V analysis, host/RF formats, worker and policy reconstruction | Complete ISA/RTOS semantics, waveform specification, calibrated RF measurements and all operating modes |
| AR9311 media | Vendor decoder command/initialization analysis, codec firmware probes, GE2D and incomplete-picture experiments | Reliable same-picture slice continuation, early-output readiness and end-to-end latency |
| AR9311 platform | Register map, GPIO readback, display wiring and selected clock/reset observations | Full silicon identification, clock ownership, RTC operation and board-specific electrical validation |
| BETAFPV P1 Air | Installed 1.0.44 identity, SC231AI/ISP findings, stock video records, NAND and calibration storage | Full stock AE/AWB, updater failure paths and RF-power calibration accuracy |
| VR04/reference air | Stock access, media/session framing, MPP/MID interfaces, panel and NT99235 findings | Sensor/board revision equivalence, unresolved frame layouts, OTA transformation and full peripheral coverage |
| Skydroid G11 | Stock platform, USB radio, GD32 images, UART/trainer formats, power/battery signals and vendor video SDK | Complete MCU-to-radio control routing, serial return wiring, physical display composition and off/wake timing |

## Evidence boundaries to retain

- Stock image hashes, installed readbacks and source SDK revisions identify
  different artifacts. A newer SDK is not proof of the installed binary's behavior.
- Original-instruction comparisons with mocked hardware establish selected
  software contracts. They do not establish RF timing, peripheral completion
  or an entire firmware implementation.
- A getter can return stale cached state. An acknowledgement can precede the
  requested hardware action or conceal a downstream failure.
- A live stream, exact decoder pixels and physical display composition are
  separate observations. Software timestamps alone are not optical latency.
- Configured RF levels and calibration indices are not measured output power.
- Matching selected partitions does not establish whole-device restoration;
  stock boots may change application or factory-volume metadata.

## Selected corrections

The [G11 stock video studies](systems/g11-stock-video-ingest.md) retain UDP
parser truncation, cache-consumption and RTSP reuse limits. The
[display lifecycle](systems/g11-stock-display-lifecycle.md) distinguishes a
destructive reset from readiness and a paused picture worker from restored
fallback output. Offline checks do not establish installed binary equivalence.

The [G11 SDK study](systems/g11-vendor-sdk.md) separates UI and HAL capacities,
corrects flattened RC structure tables and reproduces an updater wrapper that
returns success after its child fails.

The AR9311 [decoder-command](soc/ar9311-decoder-commands.md) and
[initialization](soc/ar9311-decoder-initialization.md) studies distinguish command
ordinals, cached state, allocation ownership and grouped error reporting.
[Slice experiments](experiments/ar9311-slice-decoding.md) establish reconstruction
after forced finish, not resumable decoding of one unfinished picture.

The [GPIO study](soc/ar9311-platform.md#direction-readback-correction) corrects
the early assertion that direction registers cannot be read. The
[Sky482 repair](experiments/ascent-lite-flash-repair.md) preserves the restricted
write boundary and subsequent verified stock boot/video result.

## Remaining review

The broader source and capture review remains incomplete. Future work should
expand stock teardowns, original firmware behavior, unresolved protocol fields,
and independent physical validation. Historical implementation records may
supply evidence, but replacement-stack features and deployment results do not
belong in the stock-study index.
