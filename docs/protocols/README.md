# Protocols

[Documentation index](../../README.md)

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

- [Protocol layers and recovered formats](layers-and-formats.md)
- [Ascent USB management, file transfer and information fields](ascent-usb-management.md)
- [Ascent stock backend GUI commands and derived telemetry](ascent-backend-control.md)
- [Ascent 18.21.10 GUI status and forwarded MSP](ascent-stock-msp.md)
- [Kestrel backend and passive-HUD local IPC](kestrel-local-ipc.md)
- [AR8030 host-link framing, sockets and telemetry](ar8030-host-link.md)
- [AR8030 wireless MAC and control formats](ar8030-wireless-format.md)
- [AR8030 acquisition, timing and resource management](ar8030-link-management.md)
- [AR8030 channel catalogs, measurement scores and selection policy](ar8030-channel-policy.md)
- [AR8030 internal C8 framing, events, routing and telemetry boundaries](ar8030-internal-data.md)
- [Ascent telemetry envelope and Kestrel video framing](ascent-application-streams.md)
- [P1 native UDP records and Ascent HEVC adaptation](p1-video-adapter.md)
- [System video paths](../systems/video-paths.md)

Wireless MAC fields exposed by hardware do not constitute a complete waveform or
physical-layer specification. USB and SDIO transports need separate descriptions
even where they carry related requests. Stock and replacement protocols must also
remain distinguishable.

## G11 controller interfaces

- [Stock input queries, trainer frames and physical mapping](g11-stock-inputs.md)
- [Historical custom Linux-mixer framing, sessions and CRSF forwarding](g11-custom-control.md)
- [Autonomous MCU shutdown protocol](../firmware/g11-power-and-shutdown.md#autonomous-shutdown-messages)
- [MCU presentation-event transport](../firmware/g11-feedback-and-audio.md#bounded-presentation-transport)
- [Autonomous configuration, model operations and observations](g11-autonomous-management.md)
- [MCU-owned module configuration and ExpressLRS Lua](g11-module-configuration.md)
