# AR93xx SDK changelog: API history and interpretation limits

[References](README.md) · [AR9311](../soc/ar9311.md)

An SDK changelog bundled with the extracted Ascent 17.5.8 application preserves
release headings from `sdk-ar9301-0.03.00` through
`sdk-ars31-0.16.02-08-mp` / `sdk-ar9341-0.16.02-08-mp`. The newest heading names
commit `e9e9de95f`. This is SDK history distributed in a product image, not a
manifest proving that every listed change is active in that product.

The document mixes AR9301, ARS31, AR9341 and explicitly marked AR9311 entries.
Names, capabilities and fixes must retain that scope. In particular, its
AR9341 DSP/NPU changes do not establish an AR9311 DSP/NPU implementation.
The SDK version is also distinct from the Ascent product firmware version.

## Changes useful to media reconstruction

| SDK heading | Recorded change | Why it matters when comparing binaries or APIs |
| --- | --- | --- |
| 0.03.00–0.05.00 | VI pipe frame access/RAW injection, H.264/H.265 VUI controls, ISP statistics and exposure/WB interfaces, multiprocess VPSS frame retrieval | Presence of a named API is a lead for the matching release, not a recovered ABI or proof of physical operation |
| 0.06.02 | Decoder create/start/send/get/release lifecycle; ADC raw/millivolt APIs; QSPI NAND support | Separate decoder input/output ownership and raw ADC values from interpreted measurements |
| 0.07.03 | Decoder VB-pool attachment and module parameters, bind support, VGS jobs and encoder coding controls | Shared names do not establish compatible structure sizes or buffer lifetime |
| 0.08.02–0.10.02 | Multiprocess decoder retrieval, pipeline start/stop, timestamp fixes, VDEC bind and VBR/fixed-QP encoder controls | Process/service and timestamp behavior changed across releases |
| 0.11.1 | Multiple decoder channels, JPEG decode, simultaneous MIPI DSI/DVP HDMI, GDC API rename/parameter changes | Release claims need hardware/backend gates; API names alone are not capability tests |
| 0.12.0 / 0.12.4 | Chip/board DTS separation, safe clock-rate APIs, SoC revision/ID readers, custom 3A interfaces and removal of older AE/AWB library APIs | Board descriptions and camera integrations can require migration rather than simple relinking |
| 0.13.0 / 0.13.1 | VO show/hide/priority controls, buffer changes and profile validation; 0.13.1 also lists a PTS correction | An output image or accepted setting does not establish timing correctness |
| 0.14.01 / 0.14.02 | Multipack encoded-stream retrieval, VO clock/phase controls, exposure metadata and additional board/sensor profiles | Pack boundaries and picture boundaries need separate interpretation |
| 0.15.00 | Library mode for `mpp_service`, image timestamps changed from ms to µs, keyframe-size configuration, expanded channel attributes | Service topology and timestamp units cannot be inferred from another SDK generation |
| 0.15.01 | Timestamped AE/AWB statistics, synchronized 3A frame information, stream frame-information extraction and HDMI 1080p50 | Metadata availability is distinct from calibrated timing or display support on a particular board |
| 0.16.00 | Frame insertion/removal, stream-buffer information, dynamic VPSS resolution, NV12/region changes | These are API-history leads, not evidence that every stock application exposes them |
| 0.16.02 | Dynamic decoder channel width/height plus a resolution-switch sample | Dynamic resolution in the SDK does not prove the tested receiver pipeline can change mode safely |
| 0.16.02-01-mp | Fast boot/ADB, VO frequency tracking and interrupt subscriptions, configurable exposure-table generation | Host services, display synchronization and camera policy still require matched implementation evidence |

The changelog also describes secure-boot evolution: 0.11.1 scopes its entry to
eMMC; 0.15.00 describes verification from SPL through rootfs. Neither entry
reveals the investigated unit's fuse state, keys, enabled checks or acceptance
of a modified image. The [stock updater study](../firmware/ascent-stock-updater.md)
provides the separate product-specific authentication evidence.

## Failure history is useful, but is not a diagnosis

Several entries match classes of failure encountered during reverse engineering:

- 0.15.01 lists stream-buffer-full conditions, cross-channel decoder failures,
  channel-destruction crashes and RTSP IPB decoding stalls.
- 0.15.02 removes an incorrect encoder-clock operation that prevented encoding.
- 0.16.00 lists repeated system init/exit failures, a decoder timeout/bind state
  error, encoder instability and VO start/stop problems. Its AR9311-specific
  entries include reboot, repeated GPIO interrupt enable and VO clock-divider
  calculation fixes.
- 0.16.01 lists decoder memory/stack errors, encoder channel-reset state errors
  and retention-related cleanup failures.
- 0.16.02 lists decoder exit leaks and buffer-full failures with scrambled
  stream data; 0.16.02-01-mp lists repeated-picture output after restarting the
  encoder and removes a VO CCI-related path to avoid system hangs.
- The final `-08-mp` entry lists 270-degree rotation color loss and stale UV
  data for MIPI YUV422 input converted to YUV420 output.

These entries establish that the vendor tracked such problems. They do not
show that a particular local freeze, black picture, DMA artifact or restart
failure has the same cause, or that a component in the extracted image contains
the claimed fix. The intermediate `-03` through `-07` entries largely say only
that unspecified bugs were changed; they provide no detailed compatibility map.

Retention also needs its original scope: the 0.16.00 AR9311 entry announces
DDR retention except for the network driver, and later entries fix cleanup
problems. This is not evidence of a completely qualified suspend/resume path
on Gnd485 or P1.

## How this evidence connects to the wiki

Use the changelog to locate a likely API generation, then check the matched
headers, binary callsites, structures, driver/firmware backend and experiment.
The [stock media graph](../systems/vr04-stock-media.md),
[Gnd485 stock architecture](../systems/ascent-stock-architecture.md),
[decoder experiments](../experiments/ar9311-slice-decoding.md) and
[platform validation](../experiments/ar9311-platform-validation.md) supply those
more specific boundaries. A changelog line about decoder support does not
establish same-picture incremental Wave521C decoding or a measured latency.

All 1,094 lines of the retained changelog were reviewed. This synthesis preserves
relevant API changes and limitations; it does not reproduce the vendor's full
release notes or claim that its generic sample matrix was tested here.
