# Protocols

[Documentation index](../../README.md)

- [VR04 reference MSP canvas](vr04-msp-canvas.md): serial-to-radio packing,
  shared-memory access, host capture verification and parser limits.
- [MissingLynk local messaging](mlm-ipc.md): datagram layouts, process ownership,
  recording/HUD commands and shared DRM descriptor.

Protocols are organized by the boundary they cross. Host transport, wireless MAC,
media packets and management commands are separate subjects.

| Boundary | Research findings |
| --- | --- |
| PC ↔ device management | Vendor USB serial commands, file transfer, model dispatch and update workflows |
| Linux host ↔ AR8030 | USB/SDIO loading and transport, request/reply messages and socket APIs |
| AR8030 ↔ AR8030 over RF | Acquisition, login, binding, exposed MAC fields, MCS and recovery |
| Video source ↔ consumer | Ascent backend deframing and BetaFPV-related UDP media/handshake formats |
| G11 Linux ↔ MCU | Stock UART observations and a separate custom session/control protocol |
| Overlay and telemetry | Radio state, application metadata and MSP OSD formats |

## Articles

- [Nomad USB CRSF and receiver-profile activation](nomad-usb-profile.md)
- [Protocol layers and recovered formats](layers-and-formats.md)
- [Ascent USB management, file transfer and information fields](ascent-usb-management.md)
- [Ascent stock backend GUI commands and derived telemetry](ascent-backend-control.md)
- [Ascent 18.21.10 GUI status and forwarded MSP](ascent-stock-msp.md)
- [Kestrel backend and passive-HUD local IPC](kestrel-local-ipc.md)
- [AR8030 host-link framing, sockets and telemetry](ar8030-host-link.md)
- [VR04 reference AR8030 SDIO/IP transport](vr04-sdio-transport.md)
- [VR04 media sessions, tiled HEVC and vendor receiver acceptance](vr04-video-session.md)
- [VR04 stock MID API, channel numbering and air-directed controls](vr04-stock-ui.md)
- [AR8030 wireless MAC and control formats](ar8030-wireless-format.md)
- [AR8030 acquisition, timing and resource management](ar8030-link-management.md)
- [AR8030 channel catalogs, measurement scores and selection policy](ar8030-channel-policy.md)
- [AR8030 experimental subchannels, hop confirmation and remote synchronization](ar8030-channel-integration.md)
- [AR8030 automatic scanning, measurement histories and service lifecycle](ar8030-automatic-channel-scan.md)
- [AR8030 internal C8 framing, events, routing and telemetry boundaries](ar8030-internal-data.md)
- [Ascent telemetry envelope and Kestrel video framing](ascent-application-streams.md)
- [P1 native UDP records and Ascent HEVC adaptation](p1-video-adapter.md)
- [System video paths](../systems/video-paths.md)

Wireless MAC fields exposed by hardware do not constitute a complete waveform or
physical-layer specification. USB and SDIO transports need separate descriptions
even where they carry related requests. Stock and replacement protocols must also
remain distinguishable.

Comparative study: [DJI receiver metadata, host framing and recovery contracts](dji-receiver-contracts.md).
This host-side analysis does not establish an over-the-air O4 packet format.

## G11 controller interfaces

- [Stock video-control socket, payload layouts and status limitations](g11-stock-video-control.md)
- [Stock input queries, trainer frames and physical mapping](g11-stock-inputs.md)
- [Historical custom Linux-mixer framing, sessions and CRSF forwarding](g11-custom-control.md)
- [Autonomous MCU shutdown protocol](../firmware/g11-power-and-shutdown.md#autonomous-shutdown-messages)
- [MCU presentation-event transport](../firmware/g11-feedback-and-audio.md#bounded-presentation-transport)
- [Autonomous configuration, model operations and observations](g11-autonomous-management.md)
- [MCU-owned module configuration and ExpressLRS Lua](g11-module-configuration.md)
