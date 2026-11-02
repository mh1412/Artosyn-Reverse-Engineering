# SoC and chip studies

[Documentation index](../../README.md)

| Study | Scope |
| --- | --- |
| [Artosyn Proxima-9311 / AR9311](ar9311.md) | Application CPU, memory, codec/display blocks, GPIO, clocks and platform behavior |
| [AR9311 platform registers and ownership](ar9311-platform.md) | GPIO banks and readback correction, MMC gates, watchdog, RTC and board boundaries |
| [AR9311 ADC, temperature and PWM](ar9311-analog-and-pwm.md) | Sampling/calibration, IIO units, shared registers and reference-board keypad/backlight/buzzer policies |
| [AR9311 stock GE2D](ar9311-ge2d.md) | Bitmap/RPC layouts, MMZ/cache ownership, alpha behavior and offscreen hardware copy results |
| [Artosyn AR8030](ar8030.md) | Separate radio/baseband processor, host transport and Open8030 implementation |
| [P1 camera, ISP and encoder](ar9311-p1-camera.md) | SC231AI, DMA-BUF ownership, memory pools, restart fix and automatic image controls |
| [NT99235 reference camera path](ar9311-nt99235-camera.md) | Sensor/CSI timing, VIF bypass limits, CVISP output and buffer ownership |
| [NT99235 ISP reconstruction](ar9311-nt99235-isp.md) | Register attribution, coefficient pages, statistics, adaptive ladders and corrected tone selection |
| [G11 GD32 control MCU](g11-gd32-acquisition.md) | Startup, clocks, ADC/DMA publication, battery estimation and autonomous mixer timing |
| [G11 battery and service inputs](g11-battery-and-service-inputs.md) | Charge pins, stock thresholds, voltage source validity and publication timing |
| [Wave521C slice-decoding experiments](../experiments/ar9311-slice-decoding.md) | Full-frame decoding, progressive input and incomplete-picture finalization |

Application processors, companion radios and codec processors run different
firmware. Chip studies own shared register and execution behavior; device studies
own board wiring, supplies, connectors and package identification.

Further review will expand peripheral register maps, camera interrupts,
clock/reset ownership and silicon-revision differences.
