# BETAFPV VR04 stock platform and related air-unit notes

[Devices](README.md) · [Documentation index](../../README.md)

## Identity and scope

VR04 names the goggle platform. Historical research also used `VR04-air` as a
source-target label for its associated transmitter; that label alone does not
identify a separate retail product. The investigated reference air camera was
NT99235, while the [standard P1 Air study](betafpv-p1-air.md) identified SC231AI.
Shared P1_SKY/P1_GND firmware labels do not settle the board or sensor revision.

These stock-binary and hardware findings were retained in the reference research
at kernel snapshot `e3885c6394a7025eacd3a23f473276d9823beb25` and userspace
snapshot `674ec00ecde0bba2eb844def1b4a0b7f8e685e13`. Their source provenance is
separate from proof of the installed firmware or a new hardware measurement.

The platform uses AR9311/Proxima-9311 and a companion AR8030 radio. The reference
codec image is 999,616 bytes, revision 329715; it differs from the Ascent image.
Panel, keypad, supply and radio-reset wiring must not be transferred to Gnd485.

## Goggle display and shared rail

The picture path is VO at `0x08810000` → DesignWare DSI at `0x08850000` →
QY45043A0 1920×1080 panel. VO uses GIC SPI 70, architectural interrupt ID 102;
Linux IRQ numbering is not a fixed physical identity. DSI `GEN_HDR`/`GEN_PLD_DATA`
offsets `0x6c`/`0x70` corroborate the DesignWare interface.

The vendor panel initialization has 519 four-byte DCS tuples. Its tail sends
Sleep-Out with 32 ms delay and Display-On with 40 ms delay; register `0x98`
returns panel ID `0x96`. A configuration value 1488 was interpreted as a
possible DSI bitrate, not independently established. Other named panel/bridge
variants in the binary do not identify the fitted panel.

| Global GPIO | Reference goggle role |
| --- | --- |
| 23 | AR8030 active-low reset |
| 42 | Horizontal/source-scan strap |
| 43 | Panel reset/enable |
| 95 | Second panel rail/enable |
| 100 | Shared panel-power / SD I/O 3.3 V rail |
| 107 | Vertical/gate-scan strap |

Panel bring-up drives 95 and 43 low, waits 10 ms, raises 43, waits 10 ms, raises
95 and waits another 10 ms. Orientation straps 42 and 107 must already be low
when GPIO100 rises. Leaving them floating produced the observed 180° rotation;
the correction was power-on strap ownership, not a display rotation command.

## LEDs and other hardware

The goggle has **three** addressable RGB pixels on SPI `0x01102000`, using
6.25 MHz, eight-bit transfers and GRB order. One LED data bit expands to one SPI
byte, giving 72 data bytes for the chain. Stock uses `c0` for zero and `fc` for
one, and shows orange `(G=40,R=255,B=0)`: blinking when searching, solid when linked.

The reference P1_SKY air firmware controls GPIO0 red and GPIO1 green, both
active low. Stock initialization drives both low; binding alternates `{0,1}`
and `{1,1}` at a 40 ms half-period. This mapping belongs to that studied air
board, not the goggle's SPI LED chain.

The removable-card host is `0x01c00000`, GIC SPI 49. Its I/O supply shares
GPIO100 with the panel; card-core power routing remained unresolved. The
external DS1307 at I2C0 address `0x68` kept time while powered; the removed backup
battery explained loss across power-off. Stock audio was disabled in the
investigated configuration.

## Related studies

- [Stock access and update packages](../firmware/vr04-stock-access.md)
- [SDIO/IP transport](../protocols/vr04-sdio-transport.md)
- [Media-session framing and vendor HEVC acceptance](../protocols/vr04-video-session.md)
- [Stock UI and channel controls](../protocols/vr04-stock-ui.md)
- [Stock media graph and MPP interfaces](../systems/vr04-stock-media.md)
- [NT99235 camera hardware](../soc/ar9311-nt99235-camera.md)
- [ISP reconstruction](../soc/ar9311-nt99235-isp.md)
- [Analog and PWM peripherals](../soc/ar9311-analog-and-pwm.md)
