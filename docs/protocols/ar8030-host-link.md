# AR8030 host-link framing, sockets and telemetry

[Protocols](README.md) · [AR8030](../soc/ar8030.md)

This describes the host/radio protocol recovered from stock USB traffic and
implemented by arlink/libarlink. Native SDIO carries related logical messages
with different transfer alignment. It is separate from the wireless MAC and
from the video application's own framing.

Fields verified by captured traffic and implementation are distinguished below
from observed but incompletely interpreted payloads. Values from one ground-unit
configuration are not universal RF settings.

## Logical frame

Every frame has 19 bytes of overhead:

| Offset | Bytes | Encoding | Field |
| --- | ---: | --- | --- |
| 0 | 1 | Byte | Start marker `0xaa` |
| 1 | 4 | Little endian | Payload length `n` |
| 5 | 4 | Big endian | Request ID: high-byte domain, low 24-bit subcommand |
| 9 | 4 | Big endian | Message ID for matching request/reply |
| 13 | 4 | Big-endian signed | Status; zero in the examined host request encoder |
| 17 | 1 | XOR | `0xff XOR frame[0] XOR … XOR frame[16]` |
| 18 | `n` | Message-specific | Payload |
| `18+n` | 1 | Byte | End marker `0xbb` |

The header deliberately mixes endianness. Its XOR does not cover the payload.
The parser must validate length bounds, header XOR and the end marker before
dispatching a message, and match replies by message ID rather than receive order.

USB-facing streams may split or join logical frames. The inspected library keeps
an accumulator and resynchronizes after invalid bytes. Native SDIO instead places
frames at block boundaries and treats the remainder of the last block as padding.

## Domains

| High byte | API name | Role |
| --- | --- | --- |
| `0x00` | CFG | Configuration namespace |
| `0x01` | GET | Queries |
| `0x02` | SET | Configuration/control writes |
| `0x03` | CB | Event subscription and delivery |
| `0x04` | SOCKET | Data sockets |
| `0x05` | DBG | Firmware logging |
| `0x06` | REMOTE | Remote namespace |
| `0x0a` | RPC | RPC namespace |
| `0x0b` | RPC_IOCTL | Session lifecycle |
| `0x0c` | PLAT_CTL | Platform control |
| `0xff` | Link management | Synchronization, hello and heartbeat |

Observed link-management subcommands are 1 for an initial reset/sync burst,
0 for hello and 2 for heartbeat. The recorded heartbeat interval was about
120 ms; it is an observation, not a proven minimum/maximum timing requirement.
The hello reply contains eight bytes, including a four-byte identity also seen
in status responses.

## Socket addressing and opening

Socket request IDs encode:

```text
reqid = 0x04000000 | (operation << 16) | (slot << 8) | port
```

| Operation | Meaning |
| --- | --- |
| `0x00` | Open |
| `0x01` | Write |
| `0x02` | Read/data delivery |
| `0x03` | Close |
| `0x90–0x92` | Length/transmit-limit query family |

Open supplies three u32 values: flags, transmit-buffer size and receive-buffer
size. Flag bits are RX `1`, TX `2`, reset-on-connect `4`, and datagram `8`.
Data-delivery frames use status `0x12345678`.

The examined ground application opened port 3 with RX+TX, a 2 KiB transmit
buffer and 56 KiB receive buffer for video. Port 2 used 2 KiB each way for
telemetry/application control, including payloads beginning `fe a5`.
These port assignments belong to that application, not to every AR8030 product.

## Two-stage write completion

A socket-write payload starts with an **eight-byte little-endian cumulative
accepted-byte counter**, followed by application bytes.

| Reply status | Meaning in the recovered firmware |
| --- | --- |
| `-262` | Write taken/pending; not yet permission to reuse the one-deep queue |
| Nonnegative byte count | Final accepted-byte count for that write; queue completion |
| `-263` | Previous write still occupies the send queue |
| `-264` | Supplied cumulative counter does not match the chip's counter |

An accepted write first receives a 16-byte pending response and later an
empty-payload final response. A refused write receives a 16-byte response
containing the submitted header and chip counter information.

Advancing the local counter for refused data desynchronizes subsequent writes.
Issuing the next write after only the pending reply can encounter the full queue.
Keeping the counter as 32 bits plus four zeros fails after 4 GiB; the wire field
must remain 64-bit. Transport-write success and radio-socket completion are
different events.

The stock retained-stream and worker shutdown sequence differs from the native
copy-mode endpoint. See [TX and stream ownership](../firmware/ar8030-tx-and-stream-ownership.md)
for active-write cancellation, deferred close replies and buffer lifetime.

## Recovered query and control subset

Sizes are payload bytes before logical-frame overhead. This table covers the
named subset recovered in the examined API/captures, not all possible requests.

| Domain/sub | Name | Input → output | Decoded meaning |
| --- | --- | --- | --- |
| GET/0 | `BB_GET_STATUS` | 2 → 300 | Little-endian user bitmap; `ff 03` requests `0x03ff` |
| GET/1 | `BB_GET_PAIR_RESULT` | 0 → 164 | Slot bitmap and slot-0 peer ID; unfilled bytes can be stale |
| GET/5 | `BB_GET_DISTC_RESULT` | 1 → 32 | Slot bitmap; eight little-endian i32 result positions |
| GET/6 | `BB_GET_MCS` | 2 → 8 | Direction and slot; MCS at byte 0, PHY kbps at bytes 4–7 |
| GET/10 | `BB_GET_CHAN_INFO` | 0 → 1006 | Channel-information response in the named capture |
| GET/12 | `BB_GET_AP_TIME` | 0 → 4 | Time counter; this table does not establish its unit |
| GET/107 | `BB_GET_1V1_INFO` | 4 → 216 | Frame-count argument; SNR field at bytes 0–1 |
| SET/2 | `BB_SET_PAIR_MODE` | 14 → 0 | Start/stop, slot bitmap, remaining bytes zero in the observed call |
| SET/3 | `BB_SET_AP_MAC` | 4 → 0 | Selected AP identity for DEV |
| SET/4 | `BB_SET_CANDIDATES` | 402 → 0 | Slot, count and four-byte candidate identities |
| SET/5 | `BB_SET_CHAN_MODE` | 1 → 0 | Observed 1 automatic, 0 fixed |
| SET/8 | `BB_SET_POWER` | 2 → 0 | Slot-related byte and power code |
| SET/9 | `BB_SET_POWER_AUTO` | 3 → 0 | Observed power code duplicated in the last two bytes |
| SET/12 | `BB_SET_MCS_MODE` | 2 → 0 | MCS mode |
| SET/13 | `BB_SET_MCS` | 2 → 0 | MCS selection |
| SET/28 | `BB_SET_LNA_MODE` | 1 → 0 | Startup used `ff` in the recorded configuration |

The early research contains different lengths for some initially unnamed queries;
the named table above records the later interpreted capture. A robust decoder
should verify the actual response size and firmware profile rather than infer
compatibility from a request number alone.

## Interpreting telemetry

`BB_GET_MCS` returned, for example, MCS 12 with 31,415 kbps, MCS 10 with
23,681 kbps and MCS 8 with 15,659 kbps. These are reported PHY rates, not measured
application throughput.

The first u16 of `BB_GET_1V1_INFO` correlated with application SNR using:

```text
SNR_dB = 10 × log10(raw / 36)
```

Examples were raw 6086 → about 22.3 dB and 3115 → about 19.4 dB. The remaining
216-byte structure was not fully interpreted by this capture study.

Selected 300-byte status offsets were observed as follows:

| Offset | Interpretation |
| --- | --- |
| 6–9 | Chip identity also present in hello |
| 16 onward | Record area with 12-byte stride; frequency at record offset 4 |
| 253 | MCS in the observed values |
| 254–257 | Connected peer identity |
| 266 | Power code matching observed setters |
| 268–271 | Frequency in kHz |

Some apparent two-byte fields contain a meaningful low byte followed by
uninitialized memory. Likewise, distance replies write only requested slots.
Neither padding nor unrequested records should be interpreted as fresh telemetry.

Distance values around 72–76 were observed with adjacent devices and `-1` without
a link. Indoor tests at 1, 3, 5 and 10 m showed no usable trend. The unit and
calibration are unresolved; those values must not be presented as metres.

Menu power changes produced codes `0x11` for the displayed 500 mW setting and
`0x0b` for 200 mW. This establishes a software setting correspondence, not measured
RF power or universal calibration. EV, standby and video-mode changes were
application messages on port 2 rather than direct radio configuration requests.

## Events and pairing

Subscription uses domain 3, subcommand `0x0100NN`; delivery uses `0x0200NN`.
The recorded daemon subscribed to event numbers 13 down to 0, with event 4
returning unsupported status `-2`.

| Event subcommand | Observed payload | Interpretation |
| --- | --- | --- |
| `0x020000` | Three bytes | Link-state change; slot/new/old layout remains an inference |
| `0x020001` | `00 01 NEW OLD` | MCS transition |
| `0x020002` | Four bytes | Seen near pairing/link-up; meaning unresolved |
| `0x020009` | Twelve bytes | Seen near pairing/link-up; meaning unresolved |
| `0x02000c` | Four bytes | Stepping field after link-up; meaning unresolved |

The [internal-data study](ar8030-internal-data.md#event-forwarding-and-subscriptions)
provides the complete twelve-ID size table, unsubscribe/queued-delivery behavior,
and the separate C8-to-event-8 bridge. Successful subscription does not establish
that the event producer or its physical data path has been validated.

The captured DEV bind sequence starts pairing for slot bitmap 1, polls pair
results about every half second, stops pairing, then selects the returned AP ID.
Link and MCS events follow. Remembered peer lists are host-managed and sent at
session start; successful radio discovery alone does not persist them.

See [radio loading and driver lifecycle](../firmware/ar8030-loading-and-driver.md)
for the transport and ownership rules underneath these messages.
