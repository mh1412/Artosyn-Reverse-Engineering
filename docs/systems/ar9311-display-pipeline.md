# Gnd485 stock display hardware and timing

[Systems](README.md) · [AR9311](../soc/ar9311.md)

The stock receiver's display chain is Wave521C decoder → Artosyn VO → MIPI-DSI
→ external HDMI bridge. Hardware identity, register setup and physical output
must be kept distinct from any replacement driver's implementation.

## Hardware identification and wiring

The reported physical marking is **Lontium LT9511 / C1311133B**. Software uses
an LT9611-compatible register interface; reads of registers 0/1/2 yielded
`17 02 e2`. This identifies useful compatibility without independently resolving
the package marking.

The bridge is on I²C bus 1 at address `0x3b`, through the DesignWare controller
at `0x01202000`, GIC SPI 33 and 400 kHz bus timing. GPIO107 supplies reset;
GPIO106 is the noted interrupt connection, but the examined Linux path polls
connector status. Board rail definitions are fixed-supply assumptions rather
than demonstrated Linux-controlled rail sequencing.

VO is at `0x08810000`, GIC SPI 70; DSI is at `0x08850000`. Early firmware still
establishes some pinmux, clocks and supplies. The reference goggle's panel timing
and power sequencing cannot be transferred unchanged to this HDMI board.

## Matched stock timing

The display investigation recovered this stock clock and D-PHY profile:

| Setting | Recorded Gnd485 profile |
| --- | --- |
| DC clock leaf | MIPI TX PLL / 2 / 3, low selection bits `0x702`, gate bit 12 |
| DC horizontal timing | Total 2293, sync 2037–2096 |
| DC vertical timing | Total 1125, active 1080 |
| Panel control | `0x111` |
| D-PHY PLL word | `0x044e7f72` |
| DSI | Four RGB888 lanes, approximately 928.8 Mbit/s nominal, sync-event mode |
| DSI timing | HSA 44, HBP 148, HLINE 1720 |

These internal values are board-specific programming, not proof that generic
HDMI timing parameters can be substituted. Full clock calibration, hotplug/EDID
behavior, other modes and physical camera-to-display latency remain separate
measurements.
