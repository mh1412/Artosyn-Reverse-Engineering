# AR8030 internal service construction

[Firmware](README.md) · [Internal data routes](../protocols/ar8030-internal-data.md) · [TX ownership](ar8030-tx-and-stream-ownership.md)

This records stream/queue creation, interface preflight, route construction and
resource publication recovered from the matched Ascent 18.21.10 radio. The
compared native helpers were source-only in these stages. Their semantic views
are not declarations of the original binary structures, and their test allocators
and task adapters are not a replacement RTOS.

## Stream factories and admission

The stream factory at `0x24225a` requests a 12-byte original object, initializes
references to zero and percentage to 100, then creates a queue. Byte mode uses
the requested capacity. Any nonzero retained argument instead chooses a
**100-byte queue**, five 20-byte records, and sets stream flag bit 2. Queue failure
frees the stream.

The queue factory at `0x241d1c` requests capacity plus an 80-byte header. Its
allocation wrapper requires a nonzero total **strictly below `0x8f980`**, or
588,160 bytes. This fixed-address difference is not remaining heap capacity.
The queue starts empty/enabled, then creates mutex, ready signal and space
signal in that order; each signal requests maximum 1 and initial count 0. Data
bytes are not cleared. Injected zero synchronization tokens are not rejected at
this boundary; that does not prove safe real RTOS failure behavior.

Sixty separately recorded original overflow probes wrapped capacity-plus-header
and could allocate less than the header subsequently initialized. Those used
synthetic extra backing, were excluded from matching counts and were not sent
to hardware. Native rejects the overflow. Retained mode's fixed size ignores
the unused requested byte capacity.

Low-level stream create at `0x2422d0` selects TX with flag bit 1 and RX with bit 0:

| Condition | Result |
| --- | --- |
| Invalid selector, no direction, or zero selected capacity | `-6` |
| Missing selector allocation | `-2` |
| Either requested direction already exists | `-9`, before allocation |
| TX or RX allocation failure | `-3` |

TX allocates first, with retained mode/capacity; RX always uses byte mode. Input
bits 2 and 4 set TX stream bits 0 and 1. Neither stream publishes until all
requested allocations succeed. RX failure destroys the unpublished TX object.
Existing unrequested directions survive. The original internal helper lacks
a port bound; native adds `-22` for ports outside 0–4 and missing adapters.

The host wrapper at `0x24241c` accepts selectors 0–7 for role byte 0, otherwise
only selector 0. Host ports 0–3 map to stream ports 1–4. It creates streams before
finding a handle. Under the table mutex it activates the first free record,
then unlocks before writing directions/selector/port. Publication is therefore
not one atomic mutex-protected step.

Returned handle identity comes from encoded record **bits 3–15**, not its scanned
index. Initialization must already encode the index correctly. A full table
calls stream close and returns `-11` regardless of cleanup success. Failure
can leave streams created; native does not promise rollback absent evidence.
Deterministic callback mutations establish instruction ordering, not a proven
live scheduling race.

## Preflight and built-in modes

Preflight mutates caller-owned interface/route records before role admission.
UART console and AT flags are set from configuration but not cleared on a later
call. Every route is examined, including disabled and role-ineligible records.
Identity lookup selects the first type/port/selector match and compares the full
32-bit interface selector against the route's 16-bit selector.

Source AT or console disables a route; destination AT does too. Destination
console alone does not: the original rechecks **source** console at that point.
Otherwise each found endpoint gains a reference, with self-routes incrementing
twice and a missing destination still allowing the source increment. Counts
accumulate modulo 65,536. Interfaces with no references, AT or console become
disabled. Existing disable flags are never cleared by this pass.

A missing source paired with a non-AT destination reaches an unsafe original
access. Native returns `-22` at that boundary, retaining earlier mutations and
skipping the final disable pass. These cases are deliberate guards, not original
error parity. The matched inventory contains:

| Mode | Interfaces | Routes | Action flags |
| --- | ---: | ---: | --- |
| 0 | 6 | 7 | `0x08` |
| 1 | 7 | 9 | `0x08` |
| 2 | 7 | 22 | Fifteen `0x08`, seven `0x09` |
| 3 | 5 | 6 | `0x08` |
| 4 | 4 | 5 | `0x08` |

These 29 interfaces/49 route records are a table inventory, not proof that every
record is reachable or its services run. None requests copying action bits
`0x02`/`0x04`. Mode-2 auxiliary services and the wider mode catalog still require
separate construction/runtime investigation.

## Route building and merging

The route builder consumes semantic records and an interface registry; it does
not parse loader JSON. Disabled records are skipped. Predicate flag `0x08`
requires the configured role to match. Source/second-interface IDs selected by
predicate flags must resolve below 64; actions with any `0xc8` bit also require
a destination. Pattern offset plus length must not exceed 4,114. Actions with
both `0x08` and `0x20` are rejected.

New nodes prepend. Merge requires the same full predicate/action flags and the
IDs/range/offset/length those flags select. Metadata bounds matter only when the
newly resolved source has framing enabled. Two non-obvious rules are preserved:

- Pattern **length and offset** are compared, but pattern bytes are not.
- Source IDs are compared only when predicate flag 1 is set.

A merge keeps the first pattern/filter, combines destination bits, replaces
metadata override and stores the new worker's low byte; counters survive.
Allocation failure stops without undoing earlier records. A node publishes only
after action allocation succeeds; failed action allocation leaves the unlinked
node allocated. Native adapters own that partial state rather than inventing
original rollback. Native pattern storage is bounded at 38 bytes, with extra
guards for null storage and counts above 65,535; matching original probes used
patterns no longer than 32 bytes.

## Interface type selection

The compared non-UART constructor begins at `0x210d2c`. A zero role mask admits
every role; otherwise the role selects a bit modulo 32. Missing bits return zero
before lookup/allocation. Zero therefore does not always mean publication.

Duplicate lookup compares type/port and truncates the **requested** selector to
16 bits against existing 32-bit selectors. A new object stores the full selector.
Consequently requested 65,536 conflicts with existing zero, while existing
65,536 does not block another such request at this boundary. Duplicates return
`-9996`.

Socket type 2 borrows supplied handle storage or opens a stream using the full
selector: flags `0x13` for port 0, otherwise `3`, with configured capacities.
Negative open frees the interface and returns `-9998`. Failure of the subsequent
four-byte handle-wrapper allocation frees only the interface and returns zero,
without closing the stream. Cleanup elsewhere remains unresolved.

| Interface | Selected framing/action profile |
| --- | --- |
| Socket port 0 | SBUS-related; prefix `0x0f`, trim one leading and one trailing byte |
| Socket port 1; event type 3 port 0 | C8; prefix `0xc8`, trim two leading and one trailing byte |
| Other ports | Configured framing enable, no selected custom profile |

Selected profiles force framing enabled. Socket type 2 also selects a socket
reader. Types 5/6 and out-of-table types can still publish with no writer.
These findings establish profile selection and worker trims, not a complete
SBUS parser, physical signaling or every generic framing parameter. UART type 0
remains explicitly unsupported by this native constructor (`-95`, no effects).

## Publication and task resources

After type setup, the original resource sequence is:

1. Assign/increment a 16-bit interface ID and prepend the interface to the
   registry. Later failures do not undo publication.
2. For UART AT mode, request its dedicated task at priority 3 with stack argument
   4,096, ignore the task result and return zero without ordinary reader/writer
   setup or changing enable bytes.
3. Otherwise allocate a reader, then ring metadata/payload when capacity is
   nonzero. Initialize cursors under the new mutex; payload bytes stay untouched.
4. Request the optional reader task at priority 18 with configured stack argument,
   ignore its result and prepend the reader. Partial reader allocation failure
   frees those partial allocations, records `-9999` and still proceeds to writer.
5. If a writer stack is configured, create its queue with configured capacity
   and **32-byte messages**, create its mutex and request priority-19 task.
   Queue failure yields `-9999`; writer-task failure yields `-9998` with previous
   state retained. The normal tail enables both directions, even after a reader
   allocation error, then returns that reader result.

Mutex/lock errors did not stop the tested construction path. Successful return
does not establish every requested task exists. Callers own surviving allocations
and must serialize construction. Native duplicate-publication and invalid-adapter
guards reject atomically instead of reproducing unsafe re-publication.

Constructed event port 0 and socket port 1 each request a 160-byte interface,
152-byte parser object, 20-byte ring-control object and their respective
1,043/524-byte parser storage. Socket port 1 additionally requests the handle
wrapper and a reader task with stack argument 2,048. Event port 0 has no reader
task in this configuration; host ingress feeds it directly. These are original
allocation observations, not native structure sizes or a live heap measurement.

## Evidence and limits

| Comparison | Recorded original scenarios |
| --- | ---: |
| Low-level stream admission | 1,206 |
| Host handle admission and cleanup | 789, including 32 open/lookup/close sequences |
| Stream/queue factories | 468, plus 60 separately recorded overflow probes |
| Preflight | 393 cases / 631 completed calls; six unsafe-source guard differences |
| Route constructor | 574 |
| Interface resource publication | 660 |
| Complete non-UART constructor | 694 |

Original instruction locations and relevant tables were checked against the
matched payload/container and analysis ELF. Heap, synchronization, task creation,
stream endpoints and diagnostics were intercepted where declared. Known fields
and ordered effects were compared; arbitrary padding and native layouts were not.

The route constructor required a narrow model of instruction word `0xde1b0b77`
at `0x211cd6`: extract the low word of the `s7:s6` pair shifted right by one,
preserving `s7`. Thirty-five synthetic vectors checked that model. This is not
a physical AR8030 ISA validation or evidence of a complete processor emulator.

Native composition expanded to 1,698 split/wrap cases connecting preflight,
source-interface construction, route building, parsing, dispatch and worker
delivery. Only the tested source was instantiated; other registry entries,
queues, ownership and scheduling remained adapters. These tests do not establish
full original boot, actual worker execution, teardown, concurrent pool lifetime
or live UART/SBUS service. The [runtime telemetry boundary](../protocols/ar8030-internal-data.md#runtime-boundary-and-unresolved-behavior)
records what was subsequently integrated and observed.
