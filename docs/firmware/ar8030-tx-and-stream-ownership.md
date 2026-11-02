# AR8030 transmit scheduling and stream ownership

[Firmware](README.md) · [Host protocol](../protocols/ar8030-host-link.md) · [Validation](../experiments/open8030-validation.md)

This study reconstructs the matched Ascent 18.21.10 radio's transmit path:
MCS requests, interleaver events, packet construction, stream references and
socket shutdown. Original-instruction comparisons establish selected contracts.
The later Open8030 runtime integrates only parts of them into its cooperative,
copy-based transport; the complete stock RTOS and retained-buffer lifetime are
not reproduced.

## MCS requests and interleaver events

The software bookkeeping table has **ten 12-byte records, users 0–9**. The
original initializer at `0x22520e–0x225238` stops there. The following region at
`0x26a9ac` belongs to callback-list machinery. Special hardware-register handling
for users 10/11 does not make them valid software-table indices. Earlier probes
of those indices wrote adjacent memory and were removed from supported coverage.

Initialization derives MCS from the first matching entry in a supplied 25-key
table. A request preserves an outstanding target. An increase applies
immediately; a decrease saves the prior index and starts a six-event drain.
The application callback runs on the event **after** the drain reaches zero.
Programming preserves mode transitions, code-length calculation and the
chip-specific 24-block coding adjustment. Some failures occur after a mode write
or request-cache change, so failure is not atomic.

The interleaver service at `0x227d58` separately samples two users. The first
identifies the published request; the second selects the bookkeeping record.
Callbacks can change state before later retry checks. Retry counters wrap as
bytes. Retry failure clears active state before event 9, then reports the
role-dependent user, requested/previous MCS, history and retry count minus one.
Retry count/history clear after delivery; unrelated callback changes survive.

The service also contains a distance-clock correction path: reconstruct two
timestamps, take their signed low-16-bit difference, detect a change greater
than 2,000 and schedule correction or return to neutral. Comparison covers both
scalings, signed wrap and ordering, but does not prove physical register-read
timing. Static DEV initialization supports a disabled gate on that loader path;
it is not a live measurement or an explanation of low-power acquisition failure.

Stage 200 integrated user-0 normal-key MCS drain handling. Incoming command 3
still requires slot 0, index below 16 and a supported key. Other sampled users
do not advance user 0. Stage 201 observed hardware MCS `2 → 1 → 2` with completed
drains. The exact six-event/seventh-event timing has offline and synthetic
integration evidence; the hardware snapshots do not measure every event.
Retry limits remained zero in that integration, and the original full IRQ,
queue, observer and multi-user behavior remained separate work.

## Attempt history and TX-end

Pre-encode at `0x221490` records history **before completion**, even if a later
zero budget prevents submission. It marks TX busy, samples time, posts queue 2
message `0x301` using the first user, then samples the user again for history.
An active record with nonzero retry count gets a bit. Only interleave mode
exactly 1 selects the bank-dependent bit pair. Byte subtraction wraps and shifts
use RV32 masking. Callback changes are visible when counters are reloaded.

TX-end at `0x21f778` clears busy only when its previous value equals 1, posts
queue 2 message `0x309` for the remembered user and notifies source 11. Duplicate
end events still post. Queue failure does not suppress later work. Neither
history nor this local end notification proves peer reception or acknowledgement.

The [first uplink/login trials](../experiments/ar8030-uplink-and-login.md)
demonstrate this distinction directly: three local TX completions initially
produced no AP assignment. Correcting the startup TX profile, then matching
stock startup gains, preceded the later login response.

Native guards reject unsafe table indices and initialize the original's
uninitialized notification suffix bytes. Those are deliberate differences.
The remembered TX-end user remains a full byte because that path does not index
the table. The user-0 runtime history diagnostic is described in
[runtime diagnostics](open8030-runtime.md).

## Acquiring and releasing stream storage

Registry selectors are 0–7 and stream ports 0–4; any nonzero direction selects
the alternate direction. Lookup adds a reference. Nonempty acquisition retains
it; empty acquisition drops it. Release looks the object up again and drops
both references, saturating each decrement at zero. This explains the local
acquire/release relationship, without proving allocation/teardown concurrency.

The acquisition budget is `max(1, ((request × percentage) mod 2^32) / 100)`,
with integer truncation. Comparison also executed the original floating-point
conversion/division path. Storage behavior differs:

| Storage mode | Acquisition | Release |
| --- | --- | --- |
| Retained entries | At most five spans; advance 16-bit produced offsets; pending-byte accounting unchanged | Advance consumed offsets; invoke buffer callback before removing a complete entry |
| Byte ring | At most two spans; advance acquisition cursor, subtract available bytes, add 64-bit acquired count | Consume the requested bytes when enough occupied bytes exist |

A retained entry completes only when consumed equals produced **and** produced
equals its full 32-bit length. Completed entries occupy 20-byte queue records.
Cursor/count updates precede wake. Retained accounting subtracts the requested
byte count even if traversal consumed fewer bytes; unsigned wrap is preserved.
Successful return alone therefore does not establish complete consumption.

Zero-length acquisition returns `-1` without clearing output. Missing streams
clear only the first two spans before returning `-1`. Retained success writes
five spans; byte success writes two and leaves the remaining three untouched.
Missing queues return empty success with the corresponding output extent.
Byte queues can wait when empty; finite deadlines use signed millisecond
differences and `UINT32_MAX` selects repeated 100-ms waits. Original lock/wait
results are ignored at these compared boundaries.

The producer at `0x21b500` distinguishes occupied bytes from available,
not-yet-acquired bytes. Its discard option rewinds the write cursor and removes
only available bytes from occupancy and the 64-bit written counter. Already
acquired bytes remain occupied. Discard precedes the whole-request space check
and repeats after waits, so even a `-12` rejection can have changed the queue.

Writes use up to two free spans, publish every pass and wake readers under the
mutex even when no bytes fit. The pending increment applies only after the
complete request is copied. Nonblocking partial writes return the copy count;
disable during a wait returns `-10` despite earlier publication. Null queue/data
or zero length returns `-6`. Positive deadlines use waits capped at 100 ms;
**every signed-negative producer timeout** selects indefinite retries, unlike
the acquisition helper's special handling of `-1`.

## Packet construction and retirement

Reservation selects the head slot of a ten-packet ring, advances head modulo ten
and increments ownership. A full ring asserts before these changes; a null
selected packet asserts afterward. Native returns `-95` at those boundaries
without undoing earlier reservation.

Availability scans all five ports before acquisition and reserves nothing.
Retained queues sum `length - produced` modulo 2³²; byte queues report available
bytes; negative signed totals become zero. Selection proceeds in port order,
charging four descriptor bytes per selected port. Descriptor bits 8–11 survive;
descriptors are compact while spans are stored by port. Acquisition uses timeout
zero and its return code is ignored by the original builder.

An optional policy compares a value against a word's high half multiplied by
120, then can change a control nibble and restrict selection to port 0. The
endpoints at `0x2203ca` and `0x229124` were intercepted; their units/producers
were not established as a complete backpressure mechanism.

The builder advances sequence, sets the outband flag/checksum, and submits the
header, optional outband and nonempty spans after cache maintenance. It checks
the submitted total **after segment submission**. A short acquisition can reach
an assertion without terminal submission or reference rollback. Native `-95`
preserves those partial effects. Blind retry or retirement can double-release
storage or use advertised lengths larger than those acquired.

After successful submission, selected ports 1–4 have their 64-bit acquisition
counters reset under queue/registry locks. A missing selected stream stops the
loop, retaining earlier resets; the entire mask then clears. This is accounting,
not a host completion notification. Statistics record truncated 16-bit budget
and submitted lengths, a millisecond timestamp and wrapping 64-bit totals.
`submitted - header`, modulo 16 bits, includes outband and is not necessarily
host payload alone.

Retirement at `0x230a80` releases every descriptor in each oldest packet while
owned count exceeds retention, then advances tail modulo ten. Release receives
context bits 24–27, descriptor low byte, direction zero and descriptor bits
12–31. Return values are ignored. Packet selection is held across callbacks,
but descriptor counts/words and context selector are reloaded. Caller storage
and ownership must remain stable. A guarded native error after earlier releases
cannot make retry safe automatically.

Stock pre-encode reads retention from context bits 28–31; it is not universally
one. The studied copy-mode runtime used one on pre-encode and zero on TX-end,
with no external stream descriptors to release. Agreement on zero-descriptor
ring bookkeeping does not establish equivalent buffer lifetime or backpressure.

## Early copied report and command queue

The initial native DEV queue owned ten aligned 24-byte packets: an eight-byte
MAC header and a copied 16-byte report. It later grew to ten 160-byte storage
slots, each accepting at most one copied payload descriptor of up to 132 bytes
on port 0–4. This later source checkpoint is distinct from the original
control-only trial. It does not implement stock's five-port retained-reference
scheduler.

Preparation rounds the raw hardware budget down to a multiple of eight. A
report-only packet requires 24 bytes; a payload packet requires `28 + length`.
Thus a five-byte synchronization ACK occupies 33 bytes but needs a raw budget
of at least 40. Full-ring, insufficient-budget and invalid-report failures
preserve sequence and ownership. Successful preparation advances head modulo
ten and sequence modulo 256. Caller inputs are copied; queued storage remains
immutable until the corresponding retirement event. Initialization is valid
only with hardware quiesced and no outstanding ownership.

Source 19's pre-encode handler samples user from `F70` bits 12–15, budget from
`F5C` bits 16–31 and request nibble from `F58` bits 4–7. The initial native
integration retained at most one old packet before preparing another; source
11 retired to zero. Retention one with zero owned packets is a valid no-op,
not evidence of a pending DMA. Neither preparation nor submission is completion.

Original segment submission at `0x237ca6` writes the address to `D44`, length
to `D40` bits 0–15 and a rotating one-hot index to bits 24–31. The terminal
zero-address/zero-length call resets that index and writes the index byte to
`ff`, then zero. Header, padded report/outband and payload are separate segments.
The terminal handshake's physical meaning is not independently established.
These registers are offsets from baseband base `0x0a110000`.

The command builder writes `4a`, command, payload length, XOR check and up to
128 payload bytes. It stores the check after computing it; a decompiler's alias
inside the loop was not the original instruction behavior. The encoder clears
the reserved descriptor nibble instead of reproducing allocator residue. Its
caller retains a pending synchronization ACK when budget is insufficient.

The [bounded recovery experiment](../experiments/ar8030-uplink-and-login.md#request-id-recovery-and-stream-reassembly)
toggles `B00` bit 30 while preserving this queue's ownership. Treating that toggle
as a TX-end event could reuse storage still owned by DMA.

## Closing streams and handles

Close processes TX before RX. Invalid selector/port/directions return `-6`;
missing selector allocation returns `-2`. A referenced stream remains registered:
close disables its queue, unlocks the registry, delays one scheduler unit and
retries indefinitely. Disable clears queue flag bit 1 and increments its counter
only when previously enabled; it does not wake readers or writers.

At zero references, destruction removes ready signal, space signal and mutex,
then frees queue storage unless flag bit 0 marks external storage. It clears the
queue pointer, frees the stream and clears the registry slot under the registry
lock. Occupancy alone does not prevent destruction, and retained-buffer
callbacks are not drained by this destructor. Higher layers must finish ownership.

After TX destruction, close clears its local TX selection. A new TX stream
published while waiting for RX is left alone; if TX was initially absent, it
remains selected and a later publication is visited. These ordering effects
were compared, without proving concurrent scheduler behavior.

A handle is a 12-byte record: LE flags at byte 0, selector at byte 4, port at
byte 8. Bit 0 marks active; bits 1–2 select directions. Lookup snapshots under
the table mutex. Handle close releases that mutex before draining streams, then
reads the **current** record again and clears active after success. It can return
`-1` after streams were destroyed if the record became inactive or the count
changed. Stream-close failure prevents that final update; no rollback is implied.

## Stock workers versus cooperative close

The stock dispatcher closes the handle, posts an empty TX message without
waiting and clears TX/RX running bytes, ignoring close/post results. Missing
sockets instead get an empty reply with status `0x103`. Found sockets do not
receive their final close reply immediately.

TX posts exit notification `0x0410…` **before** draining queued messages,
destroying its queue and deleting its task. Retained-mode nonempty drains adjust
pending/completion counters. RX releases any held copy buffer, posts `0x0411…`
and deletes its task. Notification failures do not undo cleanup. The dispatcher
clears each direction as notifications arrive and emits `0x0403…` after both
clear. Receiving TX exit alone does not prove that drain/task deletion finished.
Original scratch residue does not define stable unused reply fields.

Open8030 stage 206 implements a cooperative close for copy-owned buffers. It
stops new endpoint transfers, cancels partially copied writes with `-10`, keeps
completed-write results and serviced-empty-write `-7`, and preserves each write's
64-bit position and open-ID low halfword. It waits for SDIO to accept the active
completion, discards unstarted work, then keeps the close reply retryable until
accepted. Only then can the port reopen; other ports continue. Duplicate close
requests share the first pending operation.

Submitted MAC/SDIO copies are immutable and cannot be retracted by close. This
does not establish DMA completion or peer receipt. Metadata-only endpoint reset
avoids clearing large buffers in the poll loop: a rapid 18-cycle candidate test
had 87,488 dropped video bytes with full clearing; the revised workload had zero.
The accepted image passed 24 hardware cycles, including four active cancellations
and four queued discards. Stock rejected those spare-port opens because internal
streams already occupied them, so these are not matched stock close-parity tests.

## Comparison scope

The reconstruction progressed through separate, byte-bound original executions:

| Contract | Recorded comparison |
| --- | --- |
| Corrected ten-user interleaver/MCS/IRQ behavior | Replays of 7,540 interleaver, 17,496 nonfatal MCS plus 534 assertion prefixes, and 4,884 IRQ cases; unsafe historical indices excluded |
| Request/interleaver/pre-encode/end composition | 270 fresh scenarios, eight event rounds each |
| Descriptor retirement | 2,588 scenarios; 9,857 release calls |
| Stream release / acquisition | 3,600 / 3,356 scenarios, including composed ownership sequences |
| Reservation/assembly/accounting | 537 scenarios: 487 nonfatal and 50 assertion prefixes |
| Producer / close / handle close | 1,554 / 1,064 / 635 scenarios |
| Worker shutdown / write completion | 575 shutdown scenarios; 180 completion cases |

MMIO, RTOS, heap, cache, DMA submission and other endpoints were intercepted as
declared by each experiment. Synthetic memory, pointer views, ABI and partial
instruction semantics are analysis machinery. Native `-22` bounds guards,
initialized reserved bytes and assertion-to-error conversion are explicit
differences. Synthetic test counts and unchanged-runtime video captures do not
turn source-only helpers into deployed behavior. Real retained TX integration,
pending-DMA shutdown, sustained adaptation and complete scheduling parity remain
open.
