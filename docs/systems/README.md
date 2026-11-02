# Systems and applications

[Documentation index](../../README.md)

- [Video latency: DJI, Ascent HDMI and the network exporter](video-latency-comparison.md)

System studies connect the camera, encoder, radio link, decoder, display, control
and telemetry into a complete application.

## Articles

- [Architecture and video paths](video-paths.md): Ascent air/ground roles, host
  video extraction, BetaFPV-related tiled video and G11 controller integration.
- [Gnd485 stock Linux architecture](ascent-stock-architecture.md): process and
  socket ownership, Binder/CUSE services, persistent mounts and USB boundaries.
- [P1 source-stack validation](../experiments/p1-source-stack.md): source camera
  through matched radio transport, persistent startup and recovery.
- [Gnd485 decoder-to-HDMI pipeline](ar9311-display-pipeline.md): DMA buffers,
  fixed VO/DSI/bridge timing, live scanout and measured playback limits.
- [Ascent compressed-video ground station](ascent-groundstation.md): passive
  extraction, persistent RTP proxy, direct playback and video-loss recovery.
- [Ascent air/ground architecture and radio-role reuse](ascent-air-ground-comparison.md):
  host ABI differences, matched SKY/GND images and headless-receiver requirements.
- [Ascent browser interfaces](ascent-browser-interfaces.md): implemented Web Serial
  uploads, simulated-browser validation and the unresolved custom-bulk video path.
- [Open receiver stacks](open-receiver-stacks.md): historical MissingLynk/fpvOS
  boundaries and later AR9311 board-port evidence.
- [VR04 reference link control](vr04-link-control.md): association, binding,
  adaptive bitrate, power and corrected standby/recovery behavior.
- [VR04 stock media graph and MPP interfaces](vr04-stock-media.md): post-scale
  display/recording taps, service ownership and unresolved codec frame layouts.
- [VR04 open video, HUD and recording](vr04-open-media.md): tile/composite modes,
  display-buffer lifetime, FC-OSD burn-in and playback boundaries.
- [Kestrel software UI](kestrel-software-ui.md): stock video delegation, passive
  Canopy/Vector workers, input and rendering ownership.

- [G11 Linux UI, rendering and development services](g11-linux-ui.md).
- [G11 actual decoded pictures and debugger interaction](../experiments/g11-video-and-debugger.md).

- [G11 Linux model preparation, durable apply and interrupted-operation recovery](g11-model-management.md).

- [G11 Linux time, RTC and independent MCU clock](g11-clocks.md).

## Applications covered by the research

Comparative study: [DJI Goggles 3 receiver architecture](dji-receiver-comparison.md)
separates static receiver findings from encrypted air-unit internals and live
hardware claims.

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
