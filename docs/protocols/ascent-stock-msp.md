# Ascent GUI status and forwarded MSP, 18.21.10

[Protocols](README.md) · [Earlier backend framing](ascent-backend-control.md)

The October 8, 2026 investigation decoded stock Gnd485 GUI notifications
that earlier work had only counted as unmapped. The exact running `ar_ldy_gnd`
18.21.10 executable had SHA256:

```text
243c11864d266caebdc76d9c4357ca6517fd1afdbbfcbaf9d6c59c113d6d6216
```

Addresses below belong to this non-PIE binary. They must not be transferred to
the earlier 17.5.3 backend without matching the image.

## Extended receiver status

`Fxn_period_msg_to_gui` at `0x41b6bc` clears a 296-byte structure, prefixes
subtype 1 and sends **297 payload bytes** on GUI subchannel 1. Earlier 17.5.3
used a 200-byte structure and 201-byte payload. Both use the existing eight-byte
`8e 25` frame header and XOR checksum.

Verified common offsets exclude the subtype byte:

| Offset | Field |
| --- | --- |
| 0 | Quality grade, raw |
| 4 | MCS |
| 8 | Connected flag |
| 12 | Local frequency-table value, raw |
| 16 | Vendor distance estimate, raw |
| 40 | Receiver supply, millivolts |
| 100 | CPU temperature ×100 |
| 173 | Signed RF-temperature byte, raw |

The initial parser rejected the longer layout despite valid framing. Accepting
only the two verified lengths then produced 74 status updates over 30 seconds
with zero rejected/unmapped frames. Extension bytes remain opaque. Disconnected
status does not make a retained numeric MCS valid, and connected status alone
does not establish arriving or decoded video.

## Serial envelope routing

The callback at `0x41f628` routes sky message IDs 2 and 4 into FIFO 1. At
`0x41f748`, ID 2's first byte becomes 0; at `0x41f7c8`, ID 4's becomes 1.
`fxn_snd_mspdata_to_gui` at `0x420568` then sends the FIFO payload on GUI
subchannel 0 through `fpv_socket_client_send2gui_no_ack` at `0x448e40`.

| First payload byte | Meaning after removing that byte |
| --- | --- |
| `00` | Forwarded serial stream; captured MSP replies begin `$M>` |
| `01` | Separate vendor notification; contents remain opaque |

Remove this prefix once per stock envelope, not once per MSP frame. One envelope
may contain several frames, and an MSP frame can span envelopes. Prefix-1
notifications do not enter the serial parser. Other prefixes are not guessed.

## Capture and replay evidence

A three-second passive capture saved 2,332 packets / 7,923,461 pcap bytes with
zero reported socket drops. The backend-to-GUI TCP 2765 stream reassembled to
7,337 bytes with no gaps, overlaps or conflicts. Matching raw MSP bytes were
also present in the upstream radio-data flow.

Full GUI-stream replay produced seven receiver-status updates, 126 serial
envelopes, 127 MSP frames and one opaque vendor notification. There were zero
outer checksum failures, malformed MSP frames or unknown envelopes. MSP replies
comprised 118 STATUS, three ANALOG, three FC_VARIANT and three FC_VERSION.

`BTFL` identified the flight controller as Betaflight; the captured extended
version included printable `2026.12.0-alpha`. This is FC identity, not VTX
firmware identity. ANALOG reported approximately 11.5 V pack voltage and 0 A;
no cell count was present, so per-cell voltage was not inferred.

Later live logs counted 3,583 MSP frames from 3,575 serial envelopes, including
3,382 STATUS and 67 ANALOG, with zero malformed/discarded/other/request/error
counts and 29 vendor notifications. This is a longer live run, not the original
three-second replay population.

## Parsing, freshness and RC metrics

The receive-only adapter validates MSP v1 XOR and native MSP v2 DVB-S2 CRC,
direction and length. Native-v2 payloads are limited to 4,096 bytes and retained
parser storage to 4,105 bytes. Live evidence covers v1; native v2 was covered
by regression tests. Requests/error replies are counted without becoming
measurements. Valid unimplemented functions are counted as Other MSP.

STATUS and ANALOG expire independently after two seconds. Fresh arm-state or
identity packets cannot prolong stale current/voltage/RSSI. Disconnect clears
partial serial state and peer identity while retaining diagnostic counters.
The adapter does not poll the FC, open a physical UART or send setters; only
requested outer framing ACKs are emitted.

One captured ANALOG envelope was:

```text
00 24 4d 3e 09 6e 77 00 00 d1 01 00 00 a5 04 61
```

RSSI `0x01d1` is 465 of 1023, rounded to 45%. The gauge initially remained empty
because it consumed only LQ. It now selects fresh valid LQ first, then RSSI,
and labels the source `RC LQ`, `RC RSSI` or `RC --`. Genuine 0% LQ remains valid;
it does not fall back to a stronger RSSI. Later device logs showed changing
46–47% RSSI, with the corresponding half-filled overlay checked by readback.

The capture also contained vendor-batched DisplayPort warning text, but that
observation did not establish a complete DisplayPort grid decoder or an LQ feed.
Passive menu diagnostics refresh at 4 Hz to bound repaint work; message reception
and checksum validation continue at the incoming rate.

## Validation boundary

Captured-stream replay, fragmentation/corruption/expiry tests and sanitizer runs
passed. Live decoding and framebuffer readback were checked separately. A later
stock-restore/relaunch incident left the display blank/frozen and SSH without a
banner, requiring a power cycle. Its cause remained unresolved; it did not
invalidate the preceding checksum/pixel results or establish that restoration
was reliable.
