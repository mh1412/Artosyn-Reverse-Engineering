# Devices and teardowns

[Documentation index](../../README.md)

Device pages collect physical identification, board revisions, components,
interfaces, access methods and recovery history.

| Device | Article contents |
| --- | --- |
| [Ascent VRX / Gnd485](ascent-vrx.md) | AR9311 host, SDIO radio, Linux/storage architecture and USB video extraction |
| [Ascent Lite VTX / Sky482](ascent-lite-vtx.md) | Host firmware architecture, USB identities, NAND and completed startup repair |
| [Skydroid G11](skydroid-g11.md) | SigmaStar host, USB radio, MCU control and video integration |
| [BETAFPV P1 Air](betafpv-p1-air.md) | SC231AI/AR9311 hardware, physical controls, factory calibration and source firmware |

The [G11 stock platform and connector study](g11-stock-platform.md) expands the
controller overview with measured enumeration, HAL boundaries and connector orientation.

VR04-related reference hardware is covered initially in the
[system overview](../systems/video-paths.md#betafpv-related-airground-pipeline).
The tested P1 Air has an SC231AI sensor; the NT99235 reference implementation
must not be treated as an identical board.

Teardown photo sets and component maps remain to be added. Each should identify
board side/orientation and distinguish measured, traced and inferred pinouts.
