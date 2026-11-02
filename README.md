# Artosyn Reverse Engineering

Research notes on stock Artosyn-based video transmission systems: chips, boards,
firmware images, boot chains, radio and host protocols, camera pipelines, and
device teardowns. Findings distinguish binary analysis, reported configuration
and measured hardware behavior.

## Start here

- [System architecture and video paths](docs/systems/video-paths.md)
- [Ascent VRX / Gnd485](docs/devices/ascent-vrx.md)
- [Ascent Lite VTX / Sky482](docs/devices/ascent-lite-vtx.md)
- [BETAFPV P1 Air](docs/devices/betafpv-p1-air.md)
- [BETAFPV VR04 stock platform and related air-unit notes](docs/devices/betafpv-vr04-reference.md)
- [Skydroid G11](docs/devices/skydroid-g11.md)
- [Artosyn Proxima-9311 / AR9311](docs/soc/ar9311.md)
- [Artosyn AR8030 radio firmware](docs/soc/ar8030.md)

## Documentation index

| Topic | Contents |
| --- | --- |
| [Systems and applications](docs/systems/README.md) | Stock process architecture, media paths and SDK interfaces |
| [Devices and teardowns](docs/devices/README.md) | Board identification, components, wiring and access |
| [SoC and chip studies](docs/soc/README.md) | Registers, memory, codec and peripheral reconstruction |
| [Firmware](docs/firmware/README.md) | Image formats, stock binaries, boot chains, updates and recovery |
| [Protocols](docs/protocols/README.md) | USB/SDIO, radio control, media records and MCU interfaces |
| [Experiments](docs/experiments/README.md) | Stock-system probes, hardware measurements and validation limits |
| [Public references and their research scope](docs/references/README.md) | Vendor material, pinned analysis inputs and tools |
| [Research coverage and open questions](docs/research-status.md) | Documented findings and unresolved questions |

## Scope

This wiki documents stock hardware and firmware. Analysis tools, diagnostic
patches and controlled probes are included when they establish a stock protocol,
binary or hardware finding. Their behavior is identified separately from vendor
behavior. Replacement firmware, custom application development and deployment
histories are outside the documentation scope.

A shared chip or product-family name does not establish interchangeable firmware,
pinouts or protocols. Research remains incomplete; each page preserves its
board, firmware identity and evidence limits.

See the [Project structure](docs/structure.md) and
[Documentation conventions](CONTRIBUTING.md).
