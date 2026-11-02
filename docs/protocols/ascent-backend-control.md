# Ascent stock backend control and derived telemetry

[Protocols](README.md) · [Ascent ground station](../systems/ascent-groundstation.md)

The Gnd485 stock backend exposes a GUI command protocol and an existing status
feed independently of the radio's socket transport. This study concerns the
live 17.5.3 `ar_ldy_gnd`, SHA256
`80d4d78de68198e1042f4ebbf9aa33353bc21cf2c2746f85e20c9755b04a074b`.
On October 6, 2026, two read-only getters and passive status decoding were
validated. Configuration setters and the recovered keyframe trigger were traced
offline, without live execution in this investigation.

## Endpoint ownership

| Endpoint | Role |
| --- | --- |
| Backend TCP 2767 | GUI command server; one complete request per short connection |
| GUI TCP 2765 | Backend's outbound status/notification connection to the GUI |
| Backend UDP 4486 | Vendor debug commands, generally printing results to its console |
| Backend TCP 5809 | Video-file playback ingress |
| Radio TCP 50000 | Underlying vendor radio transport |
| Proxy TCP 50001 | Redirected radio transport in the ground-station installation |

Port 2765 is not a broadcast subscription service. A new client connection does
not receive a copy of the backend's existing stream. The radio proxy likewise
does not automatically see GUI telemetry. The validated observation method
passively copied the existing loopback connection.

## GUI framing

| Offset | Bytes | Meaning |
| --- | ---: | --- |
| 0 | 2 | Magic `8e 25` |
| 2 | 1 | XOR of bytes 3 through payload end, initial zero |
| 3 | 1 | Sequence, echoed in reply |
| 4 | 1 | Subchannel: 0 command, 1 status/notification |
| 5 | 1 | Bit 7 requests an ACK; response clears it |
| 6 | 2 | Little-endian payload length |
| 8 | Variable | Payload; first command-subchannel byte is command ID |

The receive buffer is `0x5000` bytes including the eight-byte header. The
server assembles a complete request, calls its handler, optionally replies,
then closes. Getter results can replace the command byte entirely. A consumer
must validate checksum, length, sequence, channel and flags, rather than assuming
every reply is a command ID followed by data.

## Validated getters

| Command | Reply | Recorded interpretation |
| --- | --- | --- |
| `0x36` | `00 00 00 00 10 10` | LE32 chip result 0, board version raw 16, RF board version raw 16 |
| `0x1a` | `02 00 00 00 00 00 00 00` | Local/air record-status and storage fields, with no active transmitter/recording |

For sequence `0x5a`, the complete requests were `8e25ed5a0080010036` and
`8e25c15a008001001a`. The chip handler ignores the underlying getter's return
code: zero is not authoritative silicon identification. Record-status and
space values remain raw until validated across actual recording/storage states.
The apparent playback getter `0x09` constructs a nine-byte response with two
zero values and does not provide a general settings readback.

## Periodic status

`Fxn_period_msg_to_gui` builds a 200-byte structure, prefixes subtype 1, and
sends a **201-byte payload** on subchannel 1. Nominal updates are about 400 ms
apart (`20 × 20 ms`), with earlier updates on MCS/link changes.

Offsets below exclude the subtype byte:

| Structure offset | Type | Meaning |
| --- | --- | --- |
| 0 | LE int32 | Quality-grade code; full enum unresolved |
| 4 | LE int32 | MCS |
| 8 | LE int32 | Connected flag |
| 12 | LE int32 | Local frequency-table value, raw |
| 16 | LE int32 | Vendor distance estimate, raw |
| 40 | LE int32 | Receiver supply voltage, mV |
| 100 | LE int32 | CPU temperature ×100 |
| 173 | int8 | RF-board temperature, raw |

A three-second capture contained 125 packets, 16,236 pcap bytes and seven
status updates, with zero reported socket drops or reconstructed-flow gaps.
It reported disconnected, 11,512 mV, 48.89°C CPU and RF temperature raw 45.

Treat RF-dependent values as invalid when disconnected **or MCS is negative**.
The [October 7 failure](../systems/ascent-groundstation.md#video-loss-and-recovery)
had connected=true, MCS=−1 and no video. Stale numeric fields must not imply a
working link. Archived subtype 3 messages carry camera/configuration notifications.
The subchannel 0 FIFO path is named `fxn_snd_mspdata_to_gui`, but that symbol
alone does not establish every payload as validated MSP or MAVLink.

Later [18.21.10 capture and replay](ascent-stock-msp.md) verified prefix-0
serial/MSP forwarding, separate prefix-1 vendor notifications and a 297-byte
periodic status payload. Those findings extend the named later firmware; they
do not make all notifications or the earlier binary's extension fields decoded.

## Offline command map and application limits

The callback at `0x419440` dispatches IDs 1–70. Representative mapped operations:

| IDs | Handler subject |
| --- | --- |
| `0a`, `0e`, `0f` | White balance, exposure, image angle |
| `0b`, `0c`, `0d`, `19` | Resolution/FPS, focus, preference, bitrate |
| `12`, `16`, `17`, `18`, `44` | Ratio, saturation, scene, sharpness, contrast |
| `13`–`15`, `03`, `23`–`27`, `29` | Recording policy/toggle and format options |
| `20`, `21`, `22`, `28`, `2a`, `2b`, `2d` | Shutter/ISO, stabilization, noise reduction, anti-flicker |
| `35`, `37`, `38`, `39`, `43` | Desired frequency, power, standby, bandwidth, frequency list |
| `42`, `46` | Video strategy, VRX UART mode |

These hexadecimal IDs are an offline map, not a validated setter API. Names can
mislead: record start/stop `01`/`02` and dynamic-bitrate enable `2e` are print-only
or lack operative setter calls in this binary; `03` invokes recording operations.

Several handlers update desired state later synchronized with the VTX. An
empty ACK does not establish successful application because operation return
codes generally are not propagated. Accepted, applied and saved-across-reboot
are separate states. GUI saved preferences need coordination with backend state;
direct configuration-file edits do not establish that either process adopted
them. A complete VTX discovery/selection contract remains unvalidated.

## Candidate keyframe request

The recovered call path is:

```text
UDP 4486, one-byte debug command 0x0a
  → fpv_video_dbg_manu_req_iframe (0x420c98)
  → request state consumed by Fxn_vdec_I_frame_req (0x420cc0)
  → fpv_video_bb_video_enable_iframe(1) (0x41d96c)
  → stock VTX message 0x20, one-byte value 1
```

This is a traced candidate for obtaining a decodable picture without restarting
the VTX. Trigger success, response time, header delivery and disconnected behavior
were not tested here. The deployed proxy does not implement this request.

A later [18.21.10 native-radio probe](../experiments/ar8030-first-video.md#host-transmit-and-a-remote-keyframe-ack)
sent the equivalent application command directly through host socket 2 and
received a matching remote ACK. That uses a different backend/address map and
does not validate this 17.5.3 UDP debug trigger. The initial acknowledged request
still did not produce a decodable picture; the video-sync sequence was separate.

A future control adapter can reuse the stock backend with individually validated
commands and explicit readback. The debug listener's console output and arbitrary
hardware operations do not constitute a general application-facing control API.
