# Systems and applications

[Documentation index](../../README.md)

System studies connect the camera, encoder, radio link, decoder, display, control
and telemetry into a complete application.

## Articles

- [Architecture and video paths](video-paths.md): Ascent air/ground roles, host
  video extraction, BetaFPV-related tiled video and G11 controller integration.

## Applications covered by the research

| Application | Relevant boundary |
| --- | --- |
| FPV air-to-ground video | Camera/encoder → radio → receiver decoder/display |
| USB/network ground station | Receiver compressed-video stream → host decoder or RTP exporter |
| Open receiver software | Application SoC drivers, media pipeline, radio interface and boot/update path |
| Integrated controller | MCU controls and mixing alongside Linux UI, video and networking |
| Video latency analysis | Capture, transport, decode and scan-out measured as separate stages |

A system page should identify the exact components and firmware combination,
show data and control paths, and distinguish a working bench setup from broader
compatibility or performance claims.
