# Experiments

[Documentation index](../../README.md)

- [VR04 reference codec, memory and DMA validation](vr04-media-validation.md):
  capability words, measured workloads, encoder reuse and later driver corrections.

Experiment pages preserve the question, setup, procedure, observations, conclusion
and limitations of a dated investigation.

## Documented investigations

- [VR04 air encoder benchmark](vr04-air-encoder-benchmark.md): camera-free load,
  packet-size limits, rate-control corrections and separate visual verification.
- [VR04 receiver latency and display pacing](vr04-receiver-latency.md): local
  timing boundaries, HUD commit stalls and conflicting pixel-clock mappings.
- [NT99235 fresh-frame proof, restart failure and ISP image corrections](nt99235-camera-validation.md)
- [AR9311 slice and incomplete-picture decoding](ar9311-slice-decoding.md)
- [Gnd485 platform, clock faults and reset recovery](ar9311-platform-validation.md)
- [Gnd485 HDMI bring-up, page-selector defect and unresolved artifacts](ar9311-hdmi.md)
- [Gnd485 SDIO radio, power/reset sequencing and packet-integrity limits](gnd485-sdio-radio.md)
- [Ascent Lite startup failure and repair](../devices/ascent-lite-vtx.md#startup-failure-and-repair)
- [Ascent Lite upload incident, restricted flash repair and runtime verification](ascent-lite-flash-repair.md)
- [Ascent compressed-video extraction](../devices/ascent-vrx.md#compressed-video-extraction)
- [Gnd485 UART0 baud limitation and scope measurements](ascent-uart0.md)
- [Ascent normal-mode file uploads, persistence and retained session state](ascent-file-upload.md)
- [Kestrel software HUD, live HDMI coexistence and renderer recovery](kestrel-ascent-ui.md)
- [Ascent proxy activation and video-loss recovery](../systems/ascent-groundstation.md)
- [Open8030 matched receiver/transmitter bench](../soc/ar8030.md#open8030-validation)
- [Open8030 validation, artifact identities and compatibility limits](open8030-validation.md)
- [Open8030 early receiver integration, loader failure and lifecycle trials](open8030-early-dev.md)
- [Open8030 early AP preparation, host-write ownership and throughput trials](open8030-ap-bringup.md)
- [P1 stock-camera to native Ascent HDMI prototype](p1-ascent-interoperability.md)
- [P1 source camera, radio and persistent boot validation](p1-source-stack.md)

- [G11 CRSF contact, electrical comparisons and fault/reset experiments](g11-crsf-electrical.md)

- [G11 decoded pixels, video/control coexistence and SWD interference](g11-video-and-debugger.md)

## Further records to consolidate

Additional device teardowns, original access
experiments and broader radio reconnect/endurance records still need review.
The detailed platform, HDMI and update pages retain the limits of the already
consolidated experiments.

Use the [experiment outline](../../CONTRIBUTING.md#experiment-page-outline).
Preserve original experiment dates, failed attempts and later corrections. A
firmware feature, a simulated test and physical end-to-end behavior establish
different kinds of evidence.

- [G11 development stages and qualification boundaries](g11-validation.md)
