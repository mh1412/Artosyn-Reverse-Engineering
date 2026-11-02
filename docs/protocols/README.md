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
- [AR8030 host-link framing, sockets and telemetry](ar8030-host-link.md)
- [AR8030 wireless MAC and control formats](ar8030-wireless-format.md)
- [AR8030 acquisition, timing and resource management](ar8030-link-management.md)
- [Ascent telemetry envelope and Kestrel video framing](ascent-application-streams.md)
- [System video paths](../systems/video-paths.md)

Wireless MAC fields exposed by hardware do not constitute a complete waveform or
physical-layer specification. USB and SDIO transports need separate descriptions
even where they carry related requests. Stock and replacement protocols must also
remain distinguishable.
