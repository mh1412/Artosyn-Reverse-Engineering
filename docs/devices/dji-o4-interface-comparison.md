# DJI O4 interfaces and product boundaries

[Devices index](README.md) · [Receiver comparison](../systems/dji-receiver-comparison.md)

## Scope

This comparison preserves the manufacturer's specification/manual snapshot checked
on **October 12, 2026**. It is not an Artosyn identification, physical teardown,
live compatibility test or decrypted O4 software study. Product documentation,
receiver firmware and inferred air-side behavior remain separate evidence layers.

## Standard and Pro

| Property | O4 Air Unit | O4 Air Unit Pro |
| --- | --- | --- |
| Published sensor format | 1/2-inch CMOS | 1/1.3-inch CMOS |
| Radio chains / antennas | 1T1R / one | 2T2R / two |
| Input voltage | 3.7–13.2 V | 7.4–26.4 V |
| Internal storage | 23 GB | 4 GB |
| Air-unit microSD | Not supported | Up to 512 GB |
| Maximum recording mode | 4K 60 fps | 4K 16:9 120 fps |
| Maximum recording bitrate | 100 Mb/s | 130 Mb/s |
| Recording container | MP4 | MP4 |
| Color modes | Normal | Normal, D-Log M |
| Published live-view rates | 1080p 30/50/60/100 fps | 1080p 30/48/50/60/100 fps |

These are the recorded [DJI specifications](https://www.dji.com/o4-air-unit/specs),
not measured results. The 100/130 Mb/s values describe recording and must not be
used as RF live-view rates. RockSteady 3.0+ and off/Gyroflow support are separately
documented stabilization choices; they do not identify the receiver's processing
graph or exact air-side buffer sharing.

The physical overview identifies a camera, coaxial cable, transmission module,
USB-C port, 3-in-1 flight-controller cable, link button/indicator and antennas,
plus the Pro microSD slot. Camera coax does not establish a particular internal
electrical protocol. The functional split is live-view encoding/radio alongside
local recording/stabilization/storage; its precise hardware and software split
remains unknown.

Receiver timing labels include `ZA530-IMX581`, `ZA530-OV48B` and `ZA5305-OV48C`.
These are configuration labels, not definitive production sensor identifications.
The series manual also covers Wide; no separate Wide archive or hardware mapping
was verified in the [package study](../firmware/dji-package-boundaries.md).

## Flight-controller interface

The retained series-manual v1.0 assigns these cable functions:

| Wire | Air-unit function | Flight-controller or supply connection |
| --- | --- | --- |
| Red | VCC | Model-specific supply range above |
| Black | Power ground | Supply ground |
| White | UART RX, 0–3.3 V | OSD TX |
| Grey | UART TX, 0–3.3 V | OSD RX |
| Brown | Signal ground | Flight-controller ground |
| Yellow | DJI HDL / S.Bus output | S.Bus input |

This is a wire-function table from documentation, not a measured connector
orientation or a pinout for another product. Canvas Mode is documented as MSP at
115200 baud, with MSP+DisplayPort for Betaflight 4.4+. Older documentation names
`osd_displayport_device = MSP` and `displayport_msp_serial`; those are historical
configuration references, not a tested setup procedure here.

The UART endpoint and goggles-side DisplayPort row/glyph/command models support
the following bounded chain:

```text
FC MSP/DisplayPort UART → documented O4 UART input
    → unresolved air-side parser and radio/message translation
    → observed goggles UI DisplayPort models → OSD rendering
```

MSP DisplayPort is an OSD command protocol, not a physical DisplayPort monitor
input. S.Bus/DJI HDL is a separate control output. The manual's sequence of
pairing Goggles 3 to the air unit and Remote Controller 3 to the goggles does not
establish the physical RF forwarding topology. The actual control relay and
air-side parser were not traced.

## Radio modes and latency claims

The recorded published operating bands are 5.170–5.250 and 5.725–5.850 GHz, with
up to 60 MHz bandwidth. The manual associates 60 MHz with Goggles 3/N3 and a
40 MHz maximum with older Goggles 2/Integra combinations. These are historical
product specifications, not a regional operating authorization or measured PHY.

Racing mode is documented for Goggles 3/N3, using manual channels, 20/40 MHz,
1080p/100 fps at 4:3, reduced transmission bitrate and disabled anti-interference
function. Up to eight channels are subject to the manual's regional constraints.

The manufacturer quotes **as low as 20 ms standard / 15 ms Pro** with Goggles 3
in that 1080p/100 fps racing mode, in unobstructed/interference-free conditions,
as a minimum during screen refresh. The study did not measure those values.
They are not typical latency, a guarantee, a per-stage budget or a measured
advantage over Ascent.

Neither these specifications nor the receiver libraries establish modulation,
FEC, retransmission policy, RF encryption or active partial-picture display.
The [receiver contracts](../protocols/dji-receiver-contracts.md) describe the
available metadata, recovery and presentation mechanisms without assigning
unobserved behavior to the air unit.
