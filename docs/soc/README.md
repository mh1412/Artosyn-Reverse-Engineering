# SoC and chip studies

[Documentation index](../../README.md)

| Study | Scope |
| --- | --- |
| [Artosyn Proxima-9311 / AR9311](ar9311.md) | Application CPU, memory, codec/display blocks, GPIO, clocks and platform behavior |
| [Artosyn AR8030](ar8030.md) | Separate radio/baseband processor, host transport and Open8030 implementation |
| [P1 camera, ISP and encoder](ar9311-p1-camera.md) | SC231AI, DMA-BUF ownership, memory pools, restart fix and automatic image controls |
| [Wave521C slice-decoding experiments](../experiments/ar9311-slice-decoding.md) | Full-frame decoding, progressive input and incomplete-picture finalization |

Application processors, companion radios and codec processors run different
firmware. Chip studies own shared register and execution behavior; device studies
own board wiring, supplies, connectors and package identification.

Further pages will cover peripheral register maps, camera CSI/VIF/ISP behavior,
interrupts, clock/reset ownership and silicon-revision differences as the detailed
findings are consolidated.
