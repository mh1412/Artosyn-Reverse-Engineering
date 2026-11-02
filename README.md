# Artosyn Reverse Engineering

Research notes on Artosyn-based video transmission systems and their applications:
chips, boards, firmware, radio and host protocols, video pipelines, and device
teardowns. This wiki connects what is inside a product with how the complete system
works, preserving the distinction between measured behavior and interpretation.

## Start here

- [System architecture and video paths](docs/systems/video-paths.md)
- [Ascent VRX / Gnd485](docs/devices/ascent-vrx.md)
- [Ascent Lite VTX / Sky482](docs/devices/ascent-lite-vtx.md)
- [Skydroid G11](docs/devices/skydroid-g11.md)
- [Artosyn AR9311 application SoC](docs/soc/ar9311.md)
- [Artosyn AR8030 radio and Open8030](docs/soc/ar8030.md)

## Documentation index

| Topic | Contents |
| --- | --- |
| [Systems and applications](docs/systems/README.md) | End-to-end video paths, ground stations, FPV and controller integration |
| [Devices and teardowns](docs/devices/README.md) | Product and board identification, components, interfaces, access and recovery |
| [SoC and chip studies](docs/soc/README.md) | AR9311, AR8030, media blocks and peripherals |
| [Firmware](docs/firmware/README.md) | Image formats, boot chains, updates and replacement implementations |
| [Protocols](docs/protocols/README.md) | USB/SDIO host links, radio control, wireless MAC, media transport and telemetry |
| [Experiments](docs/experiments/README.md) | Bench investigations, measurements, failures and corrections |
| [Research coverage and open questions](docs/research-status.md) | Established findings, validation limits and subjects still to document |

## Research coverage

| Platform | Main subjects |
| --- | --- |
| Ascent VRX / Gnd485 | AR9311, Linux architecture, USB access, video extraction and fpvOS bring-up |
| Ascent Lite VTX / Sky482 | Host/radio architecture, flash identification and startup recovery |
| Artosyn AR8030 / Open8030 | Radio firmware, host transport, link state machines and receiver/transmitter validation |
| BetaFPV P1 / VR04-related platforms | Air/ground firmware, camera and display pipelines, RF video transport and open Linux integration |
| Skydroid G11 | Linux/MCU architecture, USB-connected radio, video and EdgeTX integration |
| AR9311 media and platform | Wave521C decoding, slice experiments, HDMI, GPIO, clocks, storage and watchdog |

Results apply to the named board, firmware and experiment. A shared Artosyn chip
does not establish interchangeable firmware, pinouts or protocol behavior.
The initial pages summarize existing research; detailed register maps, protocol
specifications and teardown photo sets will expand this foundation.

See the [project structure](docs/structure.md) and
[documentation conventions](CONTRIBUTING.md) for how the wiki is organized.
