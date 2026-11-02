# AR8030 internal data framing, events and routing

[Protocols](README.md) · [Service construction](../firmware/ar8030-internal-services.md) · [Application telemetry](ascent-application-streams.md)

The matched Ascent 18.21.10 radio contains an internal graph connecting UARTs,
socket streams and host events. Its framing is distinct from wireless MAC
packets, host command framing and the Linux application's `fe a5` envelope.
Much of the graph was reconstructed through original-instruction comparisons;
the runtime integration described below is narrower than that reconstruction.

## Mode-3 graph and occupied ports

The mode-3 table contains UART2, UART1, event interface port 0, and socket
interfaces at host ports 0 and 1. Work mode and AP/DEV radio role are separate.
The socket interfaces request these streams before external host opens:

| Internal interface | Selector | Flags | TX / RX capacity | MAC stream port |
| --- | --- | --- | --- | --- |
| Socket port 0 | 0 | `0x13` | 1,024 / 1,024 bytes | 1 |
| Socket port 1 | 0 | `3` | 524 / 1,024 bytes | 2 |

All six configured route actions are `0x08`, preserving complete frames:

| Role | Bidirectional routes | Logical workers |
| --- | --- | --- |
| Common | UART1 ↔ socket port 0 | UART1 → socket uses 0; reverse uses 1 |
| DEV, role 1 | Event port 0 ↔ socket port 1 | 0 in both directions |
| AP, role 0 | UART2 ↔ socket port 1 | 1 in both directions |

Synthetic role 2 admits only the common pair. Receiver-image execution with an
AP role value is not AP hardware validation. UART1's matched configuration is
100,000 baud, eight data bits, parity value 1 and stop-bit value 3; UART2 uses
115,200, eight bits, parity 0 and stop-bit value 1. These are configuration
encodings, not new physical waveform measurements.

On stock hardware, seven ordinary opens on host ports 0/1 returned TCP status
`-259`. SDIO observation recovered inner LE32 error `-9`, which the daemon drops
when forwarding the error. RX-only and TX-only probes each failed, establishing
occupancy in both directions; this was not a small-capacity allocation limit.
Port 0 was tested at flags 1/2/3 with 64-byte capacities and flags 3 with 2,048;
port 1 at flags 1/2/3 with 64. The recovered graph supports internal occupancy, although no complete live
registry dump was obtained.
The normal backend opened port 3 at 2,048/56,320 bytes and port 2 at 2,048/2,048.

## C8 byte-stream envelope

Socket port 1 and event port 0 select this custom parser:

| Offset | Length | Field |
| --- | --- | --- |
| 0 | 1 | Sync `0xc8` |
| 1 | 1 | Length `L`, including CRC but excluding sync and length |
| 2 | `L - 1` | Body, at least one byte |
| `L + 1` | 1 | CRC8 over body only: polynomial `0xd5`, initial zero, non-reflected, no final XOR |

Total size is `L + 2`, **4–257 bytes**. The parser retains the complete envelope
at packet delivery. This recovered profile alone does not identify all bodies
or establish compatibility with an external application protocol.

The event interface's parser ring is **1,043 bytes**; the socket interface's
parser ring is **524 bytes**. The latter's separate RX queue is 1,024 bytes.
Earlier synthetic tests used 1,024-byte rings; that was not the constructor's
normal parser capacity. Normal workers reserve one byte to distinguish full and
empty. Direct original exact-fill probes made read/write cursors equal and left
input unparsed; native feed rejects that fill with `-9999` without overwriting.

The parser handles fragmented/concatenated frames and ring wrap. It separates
noise, invalid length and CRC failure into error segments, searching again one
byte after rejected sync; a damaged frame can contain the next sync. Error
counters count skipped bytes. Allocation failure consumes the chosen segment
without retry. A ring too small for a declared frame can wait indefinitely;
using a suitable constructed capacity matters.

Original comparisons covered 893 synthetic-profile cases/3,118 feeds, then
334 constructed-profile cases/2,718 feeds. The exact-fill differences were
excluded from matching counts. A composed constructor/parser/route/delivery
experiment added 108 cases/3,540 feeds, with 54 selected messages and 54 without
a route. These are bounded comparisons with allocation, queues and scheduling
adapters, not physical UART tests.

## Event forwarding and subscriptions

The event bridge clears a cached 1,024-byte buffer for each message, places a
LE16 packet length at bytes 0–1 and copies the **entire C8 frame** afterward.
The remaining bytes stay zero. Interface ports 0/1/2 emit event IDs 8/10/11.
No C8 envelope is stripped by the matched `0x08` route action.

The original returns the input length when allocation fails, even though it
emits nothing. An invalid port or injected nonzero event-hub return yields zero.
Thus route accounting can record a successful write without event delivery.
Native bounds limit input to 1,022 bytes; the original unbounded copy is not
reproduced. Producers must serialize cache use and copy notifications before reuse.

Host subscription uses `0x03010000 | id`, unsubscription `0x03000000 | id`, and
delivery `0x03020000 | id`. The original declared table is:

| IDs | Payload bytes |
| --- | ---: |
| 0 | 3 |
| 1, 2, 7, 12 | 4 |
| 3 | 322 |
| 6 | 256 |
| 8, 10, 11 | 1,024 |
| 9 | 12 |
| 13 | 136 |

Duplicate subscription succeeds without duplication. Absent unsubscription or
unknown IDs return `-2`. Notifications have zero chip message ID/status and the
declared size. Unsubscribe stops new publications but **does not cancel queued
copies**.

The comparison checked all 65,536 host IDs against the size table and 19 original
sequences/157 actions. Hardware matched 56 stock/native responses for ten
additional valid and four invalid IDs. No notifications for those extra IDs
were received by that probe, so matching subscription replies do not validate
their producers. Native nonempty-body/nonzero-status rejection is an extra guard.

## Host-to-internal ingress

Setters `0x020000c9`, `0x020000ca` and `0x020000cb` target event ports 0/1/2.
The request has operation at byte 0, LE16 outer length at byte 2 and data at
byte 4. For operation 6:

| Port | Recovered callback behavior |
| --- | --- |
| 0 | Find type-3 interface port 0; feed the outer length bytes to its parser |
| 1/2 | Require data prefix `ff a5 aa 5a ff`; take embedded LE16 length at data offset 5, add 15 modulo 65,536, and feed only if the result is at most 4,129; ignore outer length |

These data callbacks return zero even on missing interface, rejected header,
oversize or parser failure. Other operations delegate to a common control
handler. The 765 delegation cases did not establish the semantics of operations
7–10; that handler was intercepted. Missing installed core callbacks return `-4`.

Native request-extent checks reject embedded-length wrap and out-of-bounds
feeds with `-22`. Malformed requests were not sent to hardware. These setters
remained unsupported in the reviewed runtime; callback reconstruction does not
establish a live host-to-radio telemetry service.

## Route filtering and submission

An admitted route evaluates low predicate flags in order:

| Flag | Predicate |
| --- | --- |
| `0x01` | Resolve source type/port/selector and require the configured ID |
| `0x02` | Resolve source again; if its framing field is nonzero, require metadata within inclusive bounds |
| `0x04` | Resolve a second input-interface context and require its configured ID; this is not inherently a peer identity |
| `0x10` | Match configured bytes at the configured offset in the logical data span |

First matching registry entries win. Missing lookups reject. Unframed sources
bypass metadata range checks; inverted ranges reject framed input. Flag `0x08`
belongs to constructor role admission, not this predicate. The original pattern
check omitted offset from its extent test; native adds a guard. Forty-five
unsafe-offset probes are intentional differences, not matching cases.

Submission walks routes and counts every successful predicate, including later
suppressed actions. Actions with `0x08` or `0x20` share one suppression group:
after the first attempt, later matches in the group are skipped even if copying
or posting failed. Other actions continue. Flag `0x20` forces worker 0;
otherwise a logical worker byte indexes a map. An invalid worker releases the
caller reference and stops with `-4998` before that route's accounting.

Flags `0x02`/`0x04` request copies. Submission adds a queue reference, releases it
on failed post, or transfers it to the queue on success. Success decrements the
worker free counter and sends scheduler byte `0xff`; failed notification does
not undo transfer. The free counter is accounting, not queue admission, and can
wrap from zero. Attempted routes increment byte/packet counters even on copying
or posting failure; suppressed routes do not. The caller reference releases
after traversal. No match returns internal `-4996`; otherwise traversal can
return zero despite all posts failing. These are not full-parser return codes.

## Worker transformations and ownership

A received worker message first restores queue capacity under lock, accounts
incoming packets/bytes and samples time. Unsigned `current - submitted > 50`
increments an aged counter; it **does not expire** the packet. The comparison
does not establish physical clock units. Control precedence is:

| Condition | Result |
| --- | --- |
| Failed receive | `-9995`, no message accounting or ownership transfer |
| Null packet | Count missing packet, return `-9998` |
| Nonzero discard control | Release packet and return zero before action access |
| Null action | Count missing action, release packet, return `-4999` |
| Valid action | Transform, deliver, release received reference |

Transformation order is metadata override `0x100`, stripping `0x01`, envelope
prepend `0x02`, then custom-profile trim/prepend `0x04`. Default stripping
recognizes `ff a5 aa 5a ff` and attempts a 15-byte head trim. Custom stripping
requires its configured prefix, then applies head and tail trims individually;
an oversized trim is ignored. Flag `0x04` does not require prefix matching.
Missing source profiles skip profile-dependent actions. Delivery follows when
any action bit in `0xc8` is set.

The 15-byte internal envelope is:

| Offset | Field |
| --- | --- |
| 0–4 | `ff a5 aa 5a ff` |
| 5–6 | LE16 payload length |
| 7 | Metadata byte |
| 8–12 | Five zeros |
| 13–14 | LE16 payload byte sum modulo 65,536 |

Length truncates to 16 bits; the sum covers that truncated count while the
underlying payload remains whole. Extreme lengths were synthetic probes, not
normal allocation evidence. Native rejects insufficient headroom or a prefix
outside the logical span; original padded-memory probes could overwrite input
after failed prepend. Earlier transformations remain after a later error.

Direct delivery visits destination bits in ascending order across 64 positions.
Missing/disabled interfaces have separate counters. Exact returned length,
re-read after callback, counts success; short/failed writes are not retried.
Queued destinations receive an added reference and account delivery only when
their writer runs. That writer holds its mutex through receive/write/release,
does not recheck enabled state, and counts failures on the destination rather
than the route worker. Queue acceptance alone is not completed delivery.

Composed original pool tests established input recycling for non-copying routes.
Copying routes retained the allocation's initial reference after queue release,
including failed posts. In a four-block fixture, repeated action `0x0a` exhausted
copy availability while input still recycled. This does not establish a leak in
normal stock operation: none of the 49 built-in records reviewed requests the
copying bits, and dynamic reachability/other cleanup remain unresolved. Native
test adapters explicitly release their fixture-owned residual references.

## Internal socket write gate

The internal writer differs from the external host writer. Null interface/data/
handle wrapper return `-1`/`-2`/`-3`; negative handles or selector-table type other
than 2 return zero. Valid ports other than 1 delegate directly to the SDK writer.
Port 1 samples a threshold—262 bytes for mode 1, otherwise 32—then queries
`0x01000010` for selector 0, host port 1. TX-used is output word offset 96.

Nonzero role suppresses writes when occupancy is **strictly greater** than the
threshold; equality permits them. Role 0 still queries but bypasses suppression.
Suppression returns zero, which the route counts as failure for a nonempty packet,
then releases without retry. The writer forwards the original bytes, handle,
count and configured timeout when admitted. The mode's broader meaning was not
established. The compared getter clears output; a failing replacement adapter
that leaves arbitrary stack bytes is not covered. This gate and the host getter
remained outside the reviewed runtime integration.

## Evidence limits

Original-instruction comparisons cover selected parsing, routing, queue and
worker contracts with synthetic state and intercepted hardware/RTOS helpers.
They do not establish physical UART/SBUS delivery, complete concurrency or every
stock service's lifetime and teardown behavior.
