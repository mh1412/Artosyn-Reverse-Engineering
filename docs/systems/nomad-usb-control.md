# Nomad USB control and flight-controller telemetry

[Systems index](README.md) · [USB protocol](../protocols/nomad-usb-profile.md)

Scope: RadioMaster Nomad with custom ExpressLRS USB-controller firmware,
BETAFPV ELRS receivers and a HAKRCF460V2 flight controller. These retained bench
records establish a separate ground-station control path. They do not establish
Artosyn RF compatibility or resolve the
[G11 connected-module electrical failures](../experiments/g11-crsf-electrical.md).
The records identify firmware checkpoints but do not give a common experiment date.

## Hardware path and firmware progression

```text
Host CRSF sender, 100 Hz
  → CP2102 USB serial, 460800 baud
  → Nomad custom transmitter, 2.4 GHz / 500 Hz RF
  → BETAFPV Lite receiver, ELRS 4.1.0
  → CRSF UART
  → HAKRCF460V2 / G460, Betaflight 26.12.0
  → independent USB MSP channel/status readback

FC telemetry → receiver → ELRS radio → Nomad → host USB dashboard
```

The inspected Nomad uses an ESP32-PICO-D4 revision 1 and CP2102 USB bridge.
USB serial, module-bay input and backpack serial are different interfaces:
the recorded GPIO assignments are USB RX/TX 3/1, module-bay input 4 and
backpack 18/5. A working USB link does not validate module-bay wiring.

Original firmware 3.6.0 did not answer the tested USB CRSF discovery pings.
Updating to official 4.1.0 enabled USB discovery and all 34 transmitter
parameters. That stage's settings access was distinct from the later custom
USB RC implementation. The custom target omits module-bay UART/PPM setup and
uses valid USB channel frames to start and maintain the RF session.

The transmitter upgrade retained its hardware/options, NVS and original
100 mW cap; the unchanged partition layout allowed an application-only write
at flash offset `0x10000`. This cap describes the studied unit, not every Nomad.
The Lite receiver separately progressed from 3.3.0 to 4.1.0, retaining CRSF
420000-baud output and the original transmitter identity. Its recovery image
was reconstructed from the original application/options, not a full-flash backup.

The initial end-to-end custom source is checkpoint `496e82f6`, based on
upstream `4711d029`. Its reported numeric `4.0.0` comes from a nonnumeric
development version's OTA-major fallback; it is not an official 4.0.0 release.
Profile activation was tested later at firmware `9c2b4a80` and client `4fef370`.

## Observed control and loss recovery

The FC used MSP API 1.48, CRSF serial receiver input and channel map
`[0,1,3,2,4,5,6,7]`. MSP reports roll/pitch/yaw/throttle; the sender uses AETR.

| Test | Retained result |
| --- | --- |
| First valid RC stream | Established the receiver link without a handset or Bind command |
| Five axis patterns, three samples each | Roll 1250/1750 µs, pitch 1250 µs, yaw 1750 µs and neutral matched |
| Low throttle | 988 µs requested, 989 µs read back in this run; conversion rounding |
| CH6 | Both 1000 and 2000 µs read back |
| Extended channel frame | Optional arm-status byte accepted with arm flag zero |
| Concurrent settings | Power changed to 25 mW while RC remained valid; original 100 mW restored |
| Idle configuration | 25 mW read back across serial reopen/restart sequence, then 100 mW restored |
| Receiver discovery | `BFPV Lite 2G4RX`, address `0xEC`, firmware 4.1.0; positive link telemetry |
| Stop valid channels | RXLOSS absent at 0.285 s and present at 0.429 s |
| Resume fresh channels | RXLOSS cleared at the first passing sample, 1.807 s |

During the loss interval, discovery pings, bad-CRC packets and correctly
checksummed but undersized channel packets continued. They did not sustain
the link. The transmitter's **250 ms channel watchdog** and the FC's observed
RXLOSS time are separate quantities. The FC later changed its throttle readback
to its failsafe value; the transmitter does not manufacture neutral/disarm frames.

An earlier one-second recovery assertion was too short for the configured
one-second FC failsafe recovery delay plus reconnection. Polling for up to five
seconds produced the reported 1.7–1.9 s passing observations without a firmware
fix. These sampled timings are not latency guarantees. Serial reopening in the
idle-power test is also not an independently instrumented power-cycle test.

All recorded FC checks were disarmed with a temporary MSP arming inhibit during
testing. The final end-to-end status was disarmed with RXLOSS and the inhibit
cleared. No persistent FC configuration change was reported. The tested Wide
switch mode does not carry all sixteen USB inputs as independent full-resolution
RF channels, so the axis/CH6 checks cannot establish that capability.

## Receiver settings and returned telemetry

The separate receiver-settings run read all eleven Lite parameters: seven
top-level entries, two Team Race children and two hidden MAVLink entries.
It kept a 50 Hz neutral stream active while querying settings; receiver
configuration access therefore coexisted with actual FC delivery.

The dashboard received fresh battery, attitude, altitude, vario, barometer and
flight-mode groups over the physical FC → RF → USB path. Representative
comparisons were 11.6 V over CRSF versus 11.58 V over direct MSP, and attitude
approximately `(roll +2.19°, pitch −9.60°, yaw −164.40°)` versus
`(+2.2°, −9.6°, 195°)`. Heading must be compared modulo 360, accounting for
different sample times and integer MSP heading. Reported pressure/temperature
were about 973.08 hPa / 37.83 °C. Flight-mode text `AIR!` did not mean armed.

GPS was **not received** in this hardware run; its parser/display checks used
fixtures. The client tracks freshness per telemetry group, using a five-second
expiry, and marks cached data stale immediately on stop/disconnect. Missing
telemetry is not a zero reading, and satellite count alone is not a position fix.

## Profile activation and the unsuccessful two-receiver test

The physical activation test applied a temporary UID and model 17, verified
them after reboot and another USB session/reset, then restored the original
UID and model 0. The restored receiver identified itself as **BFPV Nano 2G4RX**,
with positive link quality and neutral controls at the disarmed FC. That label
differs from the Lite in the other experiments and is retained as a separate
observation. The immediate final status sample still had RXLOSS clear; it does
not independently measure the later stop timeout.

Subsequent tests attempted **B → A → B**, with both receivers reportedly
powered, 2.4 GHz / 500 Hz RF, 100 mW, model 0 and a 50 Hz neutral bench stream:

| Step | Result |
| --- | --- |
| B, manually bound to original TX identity | TX readback verified; Lite discovered; uplink/downlink LQ 100; three neutral FC samples, disarmed |
| A, phrase-derived `RX-A` identity | TX accepted and persisted settings, but no receiver responded within 12 s |
| Retry A using `RX-B` identity | Same failure after successful B baseline |
| Final B step | Not reached in either sequence |
| Cleanup | Original TX identity/model restored and verified; RC stopped; final FC disarmed with RXLOSS |

Receiver provisioning was operator-reported. Applying a UID to the transmitter
does not provision a receiver, and a successful TX Query is not proof of the
intended receiver's identity or connection. Neither run establishes two-receiver
isolation. The host's saved A profile remained on the second attempted identity;
host metadata and the restored transmitter state were different.

## Artifact identities and review limits

SHA-256 identifies the retained artifacts; images and backups are not included
in this wiki. The configured images preserve options specific to the tested unit.

| Artifact | Bytes | SHA-256 |
| --- | ---: | --- |
| Original 3.6.0 full transmitter backup | 4,194,304 | `ac794c4b1b881de8ad7274a7a6064722d3dac84801571e1f67538cd497998411` |
| Official 4.1.0 configured transmitter application | 1,697,376 | `352274672896964fa55933735dafb64163789edbf83e869a1c9f9d0ae8b6d857` |
| Initial custom USB application | 1,690,128 | `bbc9fb5edd2ac932e63bd09c8fe412e9b983320d034a9531ebecfb5532928606` |
| Profile-capable custom application | 1,692,224 | `3ce9247f8df0f0043cd35dd0a9b42b8d9585b02b563a4d39c683f9b8604fa09c` |
| Configured Lite 4.1.0 receiver application | 550,447 | `634c8dc4a28e7e28bc54a6b8b6cd51c5eb1a68ca21494317e4d0d4901d59fa8a` |

Documentation review rechecked these hashes, fifteen axis samples, loss/recovery
records and seven retained CRSF frame checksums. Seven native parser/watchdog
tests and sixteen Python protocol/activation tests passed afresh. The latter
use simulated transports, including disconnect and wrong-readback cases.
Hardware was not rerun. Range, optical video latency, full channel resolution,
two-receiver isolation and G11 module delivery remain outside these results.
