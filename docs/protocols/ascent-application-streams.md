# Ascent stock application messages

[Protocols](README.md) · [Host link](ar8030-host-link.md)

These formats sit above the radio's opaque socket transport. In the studied
Ascent receiver, MAC port 3 maps to host socket 2 for application messages.
Linux, rather than the AR8030 firmware, owns telemetry decoding and UI freshness.

## `fe a5` application envelope

The recovered stock envelope has these fields:

| Offset | Bytes | Field |
| --- | ---: | --- |
| 0 | 2 | `fe a5` |
| 2 | 1 | Sum of bytes 3, 4 and 5 modulo 256 |
| 3 | 1 | Sequence |
| 4 | 1 | Subchannel bits 0–2, ACK bit 3, payload-length low nibble in bits 4–7 |
| 5 | 1 | Payload-length upper eight bits |
| 6 | `n` | Payload; application type first |
| `6+n` | 1 | Trailer, opaque in the recovered stock format |
| `7+n` | 4 | Little-endian CRC32C |

```text
n = (byte[5] << 4) | (byte[4] >> 4)
total length = n + 11
```

The 12-bit payload length permits at most 4,095 bytes, giving a 4,106-byte maximum
envelope. CRC32C is reflected polynomial `0x82f63b78`, initial zero, **without
final complement**, covering the complete six-byte header, payload and trailer.
A standard library CRC32C function with different initial/final conventions is
not automatically equivalent.

Header checksum and length can be checked before the complete payload arrives.
The stream parser must retain incomplete envelopes, verify CRC and resynchronize
after invalid data; one socket read is not necessarily one application message.

## Keyframe and video-session commands

The matched 18.21.10 backend sends subchannel-zero payload `20 01` to request
a keyframe. A native socket probe received a CRC-valid reply with the same
sequence/subchannel, ACK bit set and payload `20`. The original ACK dispatcher
matches the pending request by those fields. This is remote application receipt;
the capture still lacked parameter sets and decoded video at that stage.

A separate `28 00` video-sync message is sent ten times at a 300-ms timer
interval after the backend's link-up event. Both messages use extension/trailer
zero in this matched sender. Local send flags are not automatically that wire
byte. The original backend also repeats keyframe enable at 800-ms intervals
while awaiting decoder readiness; the early probe sent it only once.

Command acceptance, parameter sets and a fresh decodable picture are separate
observations; neither command alone guarantees usable video.

## Telemetry semantics

Type 5 carries transmitter status. Recovered host consumers interpret payload
byte 1 as signed temperature in °C and bytes 11–14 as LE32 supply millivolts.
Type 2 carries flight-controller bytes; captures included MSPv1 and native MSPv2.

| Observed MSP command | Meaning |
| --- | --- |
| 2 | Flight-controller firmware identifier |
| 3 | Firmware version |
| 101 | FC status, cycle time, errors and sensor/mode masks |
| 110 | FC analog/battery values |

Transmitter supply and FC battery voltage are different measurements. Decoded
values are not independently calibrated sensor readings, and mode bits require
the active box mapping. Wire presence does not guarantee display in every stock
UI or define freshness after a link interruption.

These fields do not establish RC uplink/failsafe, MAVLink, physical UART/SBUS
routing or every stock UI field. The [C8 internal service](ar8030-wireless-format.md#port-0-commands-and-c8-service-stream)
is a distinct path.
