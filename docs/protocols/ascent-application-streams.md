# Ascent application messages and Kestrel video framing

[Protocols](README.md) · [Host link](ar8030-host-link.md)

These formats sit above the radio's opaque socket transport. In the studied
Ascent receiver, MAC port 3 maps to host socket 2 for application messages.
Linux, rather than the AR8030 firmware, owns telemetry decoding and UI freshness.
The Kestrel video extensions below are explicitly identified; they must not be
attributed to stock firmware or assumed available on every backend.

## `fe a5` application envelope

The recovered envelope and Kestrel encoder/parser agree on:

| Offset | Bytes | Field |
| --- | ---: | --- |
| 0 | 2 | `fe a5` |
| 2 | 1 | Sum of bytes 3, 4 and 5 modulo 256 |
| 3 | 1 | Sequence |
| 4 | 1 | Subchannel bits 0–2, ACK bit 3, payload-length low nibble in bits 4–7 |
| 5 | 1 | Payload-length upper eight bits |
| 6 | `n` | Payload; application type first |
| `6+n` | 1 | Trailer, opaque in the recovered stock format; Kestrel writes zero |
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
Kestrel's examined encoder writes subchannel zero and sets ACK for the acknowledged
sequence. This does not define every stock subchannel or retransmission policy.

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

An experimental radio-side decoder and telemetry diagnostic were removed in
Open8030 stage 232. The intended division keeps radio framing/socket delivery on
AR8030 and application/MSP interpretation in the host. Successful telemetry does
not establish RC uplink/failsafe, MAVLink, physical UART/SBUS routing or every UI
field. The [C8 internal service](ar8030-wireless-format.md#port-0-commands-and-c8-service-stream)
is a distinct path.

## Video envelope

The Kestrel protocol declares a stock-derived video envelope with extensions in
previously zero fields:

```text
00 00 00 01 | 42-byte header | 12-byte extras | encoded payload | ee 29 55 9f
```

Total overhead is 62 bytes. “Slice” in the format names the packet payload;
the P1 backend currently supplies complete-frame HEVC and does not thereby
implement partial-frame output.

All offsets below begin at the first header byte, after the four-byte start code:

| Header offset | Bytes | Field |
| --- | ---: | --- |
| 0 | 1 | `0x80` |
| 1 | 1 | Sum of header bytes 2–41 modulo 256 |
| 2 | 1 | Kestrel fractional AP-clock byte when advertised, 1/256 ms |
| 3 | 1 | Picture type |
| 4 | 4 | LE32 encoded payload length, including appended SEI |
| 8 | 1 | Unspecified here |
| 9 | 1 | Encoded frame rate |
| 10 | 1 | Kestrel protocol version |
| 11 | 1 | Kestrel feature bitmap |
| 12 | 2 | LE16 picture number |
| 14 | 2 | LE16 width |
| 16 | 2 | LE16 height |
| 18 | 8 | LE64 capture PTS, microseconds on the host/media clock |
| 26 | 4 | LE32 radio clock, milliseconds |
| 30 | 1 | Capture-to-packet-build delay, milliseconds |
| 31 | 1 | Extras length, 12 in the declared format |
| 32 | 1 | Slice index bits 0–4, MORE `0x20`, LAST `0x40` |
| 33 | 1 | Kestrel tag `0x4b` (`K`) |
| 34 | 1 | Configured frame rate |
| 35 | 2 | Kestrel capture-to-encoder-output time |
| 37 | 2 | Kestrel packet-ring waiting time |
| 39 | 2 | Kestrel previous-payload radio-write time |
| 41 | 1 | Kestrel queued payload depth, clipped to 255 |

The three latency fields are LE16 in 10 µs units, clipped to 65,535. The radio
clock timestamp is specified at encoder output when APCLOCK is advertised.
Capture PTS and radio time are different clocks; subtracting them without an
established mapping does not measure end-to-end latency. The write-time field
describes the previous payload, not the current one's future completion.

The extras contain two six-byte records: `06 02` plus the payload's LE32 XOR,
and `06 01` plus the cumulative transmitted-byte field. Payload XOR combines
little-endian u32 words and then XORs each remaining byte. It is separate from
the header sum and the application envelope's CRC32C.

## Kestrel feature identification

Protocol version is 1. The version message (type `0x04`) uses payload bytes
1–2 for `KA`, byte 3 for protocol and byte 4 for feature bits. The video header
repeats identifying information because version messages are not emitted in
every state.

| Bit | Declared feature |
| --- | --- |
| `0x01` | Latency fields |
| `0x02` | Intra refresh |
| `0x04` | Camera IMU carried as SEI |
| `0x08` | Encoder-output AP-clock timestamp |
| `0x10` | Ground-requested bandwidth cap |
| `0x20` | FC IMU via polled MSP |
| `0x40` | 1536×864 at 120 fps mode |

These are protocol definitions, not a capability list for the SC231AI P1 backend.
Use the advertised bits, backend implementation and its hardware evidence
together. Stock zero-filled extension positions must not be interpreted as
Kestrel timings or feature support.

## Additional Kestrel messages

Type `0x50` supplies newline-delimited `key=value` identity/status text. Keys
can include application/version, OS, radio, kernel, stock baseline, board/model,
sensor, readout and exposure. Unknown keys are skipped and absent keys remain
unknown.

Type `0x51` describes radio-frame timing: generation byte, LE32 duration in
µs×256, LE32 modulo-frame end of the video interval in the same units, and a
one-byte radio-frames-per-picture value (zero when unavailable). These units
differ from the video header's 1/256 ms fraction.

Ground-to-air extensions are `0x40` for a LE32 bitrate cap in kbps (zero means
none), `0x41` for bandwidth cap (20 MHz, with 40/0 meaning none), and `0x42` for
flight-log enable. Their definitions do not establish that every board implements
all corresponding hardware behavior.
