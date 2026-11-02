# AR8030 wireless MAC and control formats

[Protocols](README.md) · [Link management](ar8030-link-management.md)

Scope: the matched Ascent 18.21.10 DEV receiver reconstruction, principally
stages 216–234. Later AP firmware and P1 integration are separate validation
stages. These are MAC bytes and control fields exposed by radio hardware,
not a complete waveform, preamble, coding or interleaving specification.

## Namespaces

| Term | Meaning |
| --- | --- |
| FCH | Hardware frame-control header and coding/control observations |
| MAC | `0x58` frame, descriptors, outband and concatenated payloads |
| Outband | MAC control region, not a separate RF channel |
| AP control | Coordinator's full or compact broadcast |
| DEV report | Receiver's distinct uplink status format |
| MAC port 0 | Internal command byte stream |
| MAC ports 1–4 | External streams mapped to host sockets 0–3 |
| Host frame | Separate `0xaa…0xbb` Linux/radio framing |

Four-byte identities retain wire order. Channel bytes are indices into a
configured frequency list. Peer slots, logical PHY users, hardware buffer users
and socket ports are independent namespaces. Logical receive user 8 mapping to
buffer user 0 does not make either the assigned peer slot.

## MAC packet

Multi-byte fields here are little-endian:

| Offset | Bytes | Field |
| --- | ---: | --- |
| 0 | 1 | `0x58` |
| 1 | 1 | Zero; other formats unsupported by this parser |
| 2 | 2 | Header/descriptor CRC16 |
| 4 | 1 | MAC sequence, wrapping modulo 256 |
| 5 | 1 | Opaque; studied DEV encoder writes zero |
| 6 | 1 | Unpadded outband length `O` |
| 7 | 1 | Descriptor count `N` |
| 8 | `4N` | Descriptors |
| `H = 8 + 4N` | `O` | Outband |
| `H + O` | 0–3 | Alignment padding |
| `H + align4(O)` | Variable | Payloads, in descriptor order |

Each descriptor is:

```text
word = (length << 12) | (flags << 8) | port
       20-bit length    4-bit flags   8-bit MAC port
```

For example, `03 a3 00 00` describes port 3, flags 3 and ten bytes. General flag
semantics are unresolved; the studied DEV transmit encoder writes flags zero.

CRC16 starts at zero, processes MSB-first with polynomial `0x1021`, has no final
XOR, and treats bytes 2–3 as zero. It covers exactly `H` bytes: header and
descriptors, **excluding outband and payload**. Store the result little-endian.

The decoder bounds every extent before exposing payload views. Trailing DMA
padding is excluded from decoded length. Although the fields permit 255
descriptors and 20-bit lengths, actual buffers are smaller. The matched encoder
limits outband to 252 bytes because a padded extent of 256 is unsupported.
Exposing the sequence byte does not implement generic duplicate suppression or
per-packet acknowledgement.

## Full AP control

The parser accepts a 32–200-byte extent. Byte 0 declares a length of at least
32 that must fit the supplied extent.

| Offset | Bytes | Field |
| --- | ---: | --- |
| 0 | 1 | Declared control length |
| 1 | 1 | Control sequence, distinct from MAC sequence |
| 2 | 1 | Lookup-based checksum |
| 3 | 1 | Mode bits 0–1, relay bit 5, acquisition-related bit 6 |
| 4 | 1 | Slot bitmap |
| 5 | 1 | Uninterpreted |
| 6 | 1 | Receive-channel index, implementation naming |
| 7 | 1 | Transmit-channel index |
| 8 | 4 | Peer identity |
| 12 | 16 | Eight LE16 slot records |
| 28 | 4 | Extension bitmap |
| 32 onward | Variable | Extensions in increasing bit order |

The studied connection requires both channel indices to match its current
channel. It does not establish an independently negotiated uplink frequency.
Slot-record low bits have context-dependent availability/state meanings, and
the high byte is an assignment token; do not interpret overlapping bits as
independent Boolean properties.

Only extension bits in mask `0x2f` are accepted:

| Bit | Extent | Meaning |
| --- | --- | --- |
| 0 | Bitmap byte plus ten bytes per set slot | Packed login records |
| 1 | Bitmap byte plus one byte per set slot | Bounded per-slot values |
| 2 | Count byte plus four bytes per address | Address list; locate local identity |
| 3 | Count byte plus that many bytes | Bounded byte view; general effects unresolved |
| 5 | One byte | Bounded value; general effects unresolved |

For extensions 0 and 1, an empty bitmap has the matched reader's unusual
zero-byte extent: it examines that byte without advancing over it. Bit 4 and
unknown higher bits are rejected. Parsing an extension is not proof that its
runtime effect is implemented.

Each ten-byte login record contains token at 0, a retained login value at 1,
receiver identity at 2–5 and remote synchronization bytes at 6–9. Its slot is
determined by the bitmap. For bits 0 and 3, the two records describe slots 0
and 3, not slots 0 and 1. The token must match the corresponding slot record.

## Compact AP control

This form is exactly six bytes:

| Byte | Field |
| --- | --- |
| 0 | Lookup checksum of bytes 1–5 |
| 1 | Sequence |
| 2 | Assignment token |
| 3 | State bits 0–1; slot flag bit 2 |
| 4 | Status bits 0–2; power-index low five bits in bits 3–7 |
| 5 | Power-index bit 5 in bit 0; remaining bits uninterpreted |

It has no peer identity, channel list, mode or slot bitmap. The studied DEV
path applies it only to an existing state-2, slot-0 connection. A changed token
or non-2 state invalidates that connection; compact control cannot admit a new
peer. Decoding a power-index field is not dynamic power-control validation.

## Release-specific lookup checksum

Both AP forms use the matched lookup behavior, not an arbitrary CRC-8 routine.
For indices below 248, XOR the basis value for each set index bit:

```text
bit:    0  1  2  3  4  5  6  7
basis: 04 08 10 20 23 25 29 31
```

Indices 248–255 return zero. Start accumulator zero and update with
`step(accumulator XOR byte)`.

- Full form: process declared length, substituting zero for byte 2; compare
  with stored byte 2.
- Compact form: process bytes 1–5; compare with byte 0.

## DEV uplink report

The receiver's 16-byte report is not the AP full format:

| Offset | Bytes | Field |
| --- | ---: | --- |
| 0 | 1 | Length 16 |
| 1 | 1 | Zero |
| 2 | 2 | CRC16 over all 16 bytes, checksum bytes zeroed |
| 4 | 1 | State bits 0–1; assigned slot shifted left two |
| 5 | 1 | Receive-channel index |
| 6 | 1 | Six-bit slot value and two raw status-derived bits |
| 7 | 1 | Zero |
| 8 | 4 | Local identity |
| 12 | 4 | Zero in the no-extension report |

Slot **8** is the unassigned sentinel, not a ninth usable slot. State 0 reports
slot 8. During acquisition, transmit context 0 and assigned slot 8 coexist
until login assigns slot 0. The initial power index supplies the assigned
slot's six-bit value; the high status bits remain uninterpreted.

The compared encoder accepts state 0–2 and a four-bit slot input. State 0
forces slot 8 regardless of that input. Slot 8 encodes value zero; other slots
below 12 use the supplied six-bit value; slots 12–15 encode 63. These raw
encoding cases do not establish additional usable peer slots beyond 0–7.

Synthetic test vector with invented identity `01 02 03 04`:

```text
58 00 cd d6 00 00 10 00
10 00 a0 fc 21 02 40 00 01 02 03 04 00 00 00 00
```

It contains MAC sequence 0, no descriptors, state 1, unassigned slot 8 and channel
index 2. Header CRC `d6cd` and report CRC `fca0` are independent. This is an
offline encoding example, not a captured over-air packet.

The [stock report comparison](../experiments/ar8030-uplink-and-login.md#a-stock-report-reference-with-a-corrected-status-claim)
later reproduced both state-1/slot-8 and state-2/slot-0 reports byte-for-byte.
Its raw status inputs 2/2 reproduced the two high bits of byte 6, but did not
uniquely recover those raw bytes. The compared transformation is
`(((raw_status XOR 2) >> 1) & 1)` for each bit. Those fields retain unresolved
physical semantics; they are not calibrated gain readings.

## Separate mode-3 PHY control block

The original builder at `0x229ee6` creates an 18-byte DEV control block, written
through `0x24b514` to baseband offset `B8C` onward. Its caller at
`0x22fa3c–0x22fa82` explicitly gates this path on mode 3. In pre-connected
states it includes local identity and receive metadata; the metadata helper
rejects samples older than 200 ms.

Its check clears **byte 0**, calculates CRC16 over all 18 bytes and stores the
low byte. This differs from both MAC/DEV-report CRC16, which zeroes bytes 2–3,
and the lookup-based full AP control check. It is an original-code contract,
not an integrated or hardware-qualified native mode-3 transmitter.

Mode-zero startup instead initializes seven bytes, seals them with the FCH
lookup convention and writes `1e 00 00 00 00 00 00` as two padded words at
`B8C/B90`. The 18-byte mode-3 block is not a mode-zero startup dependency.

## Port-0 commands and C8 service stream

Port 0 is a byte stream: envelopes can span descriptors/packets or share one
descriptor. Its envelope is `4a | command | length | xor | payload`, with payload
length at most 128. XOR of the whole envelope must be zero. The parser retains
partial envelopes and resynchronizes after noise/checksum failure.

Command 4 emits a one-byte lookup checksum of the four login synchronization
bytes. It is not an acknowledgement of every video packet. The studied DEV TX
path carries a 16-byte report plus at most one copied payload descriptor of 132
bytes. Report-only length is 24; adding a payload uses `28 + payload_length`.
Available hardware budget is rounded down to a multiple of eight. Synchronization
ACK has priority over host bytes, including when the current budget is too small.

A separate internal socket-port-1 format is `c8 | L | body | CRC8`. `L` counts
body plus CRC; total is `L+2`. Body length is 1–254. CRC uses polynomial `0xd5`,
initial zero, non-reflected bits and no final XOR, over the body alone. The
parser retains the full envelope, handles fragmentation/ring wrap and sends
malformed bytes through a separate error path.

The recovered mode-3 graph reserves host ports 0/1 for internal UART/SBUS/event
services; AP and DEV attach different physical services. A fixed C8 receive
bridge produces event 8, with a 1,024-byte body containing LE16 length, payload
and zero tail. It does not establish full bidirectional service provisioning.

FC data also appears in the independent Linux application stream on MAC port 3 /
host socket 2, including MSP. No C8 packets therefore does not imply no telemetry.

## Seven-byte PHY control

The hardware exposes another control message beginning at bit 28 of sampled
control-header words:

| Byte | Field |
| --- | --- |
| 0 | Lookup checksum |
| 1 | Command low nibble, TTL low four bits in high nibble |
| 2 | TTL bit 4 in bit 0; payload length in bits 1–3 |
| 3–6 | Payload storage, including retained/unused tail |

Checksum processes bytes 1–6, with byte 6 ORed with `0xf0`. Unused tail bytes
still count. Seven zero bytes fail: the computed checksum is `0x1e`.

This register-level check is separate from MAC integrity and the full-control
checksum. In the [first native DMA trials](../experiments/ar8030-receive-acquisition.md#trials-95-and-96-independently-valid-control-packets),
FCH diagnostic checks failed while independently retained MAC/control bytes
passed their own checks. Conversely, earlier hardware-valid indications had
produced no DMA and invalid extracted controls. Neither observation can replace
validation at the other layer.

| Command | Payload length | Effect in the studied DEV runtime |
| --- | ---: | --- |
| 0 | No fixed check | Cache/observation |
| 1 | 1 | Effect unsupported |
| 2 | 4 | Effect unsupported |
| 3 | 2 | Limited transmit-MCS update; outgoing receive-MCS request format |
| 4 | 3 | Effect unsupported |
| 5 | 3 | Effect unsupported |
| 6 | 2 | Selected RX/TX one-byte PHY identities |
| 7 | 4 | AP-time offset |
| 8 | 4 | Effect unsupported; byte 6 bits 2–3 must be clear |
| 9 | 0 | Cache/timer handling, no original callback |

Commands above 9 are rejected. Command numbers in this layer are unrelated to
same-numbered host or port-0 commands. See [timing and state management](ar8030-link-management.md)
for TTL, clock, MCS and recovery behavior.
