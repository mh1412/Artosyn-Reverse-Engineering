# G11 autonomous configuration and observation protocol

[Protocols](README.md) · [G11](../devices/skydroid-g11.md)

This interface manages MCU-owned control. It reuses COBS/version-1/CRC16 framing
and the 64-byte payload limit from the
[custom transport](g11-custom-control.md), but has its own management ownership
and model operations. Legacy Linux-supplied channel packets are not its control
input. Configuration uses types 24/25; model operations use 26/27. Shutdown
(28/29) and calibration (30/31) are separate services on the same connection.

## Ownership, ordering and retry

HELLO reads the durable boot nonce, connection generation and store status.
OPEN echoes that identity with a nonzero host session, increments connection
generation and starts request sequence 1. Subsequent sequence numbers are
contiguous and cannot wrap; sequence exhaustion requires OPEN, generation
exhaustion a new boot. Model activation generation is a separate counter.

Only the most recent accepted request has an exact-retry cache. Packet type,
body and envelope must match; altered content at the same sequence is refused.
Retries return the original reply, including errors, without repeating side
effects. An admitted refusal still consumes its sequence. Changed circumstances
require a new request. HELLO preserves retry state. Cached observations need not
reflect current coordinator state.

OPEN, silence and disconnect neither stop control, abort a candidate, change the
model nor enable output. There is no host heartbeat dependency. Expensive cache
work checks an independent quiescence gate which only the MCU coordinator can
establish. Boot/session/generation tokens fence stale traffic; they do not
authenticate a host.

## Candidate transfer

All integers are little-endian. The enclosing packet's `sample_us` is zero.
HELLO is session/sequence zero with two bytes: version 1, operation 0. Other
requests have version at 0, operation at 1, boot nonce u64 at 2, connection
generation u32 at 10, then the body at 14. OPEN uses sequence zero.

| Operation | Value | Body |
| --- | ---: | --- |
| HELLO | 0 | Special two-byte request |
| OPEN | 1 | Empty |
| STATUS | 2 | Empty |
| BEGIN | 3 | Transaction u32; schema u64; model u64; revision/size/CRC32 u32 |
| WRITE | 4 | Transaction u32; offset u32; 1–42 data bytes |
| FINISH | 5 | Transaction u32 |
| STEP | 6 | Transaction u32 |
| ABORT | 7 | Transaction u32 |

WRITE uses even offsets, with odd length only at the payload end. STEP performs
one erase/verify/commit step, never an implicit polling repeat. FINISH initiates
CRC/semantic verification. READY is a durable candidate, not an active model.
ABORT clears candidate bookkeeping without erasing flash or changing selection.

Replies are exactly 64 bytes and echo session/sequence:

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0–3 | 1 each | Version / operation / result / store state |
| 4 | 8 | Boot nonce |
| 12 / 16 / 20 | 4 each | Connection generation / transaction / received count |
| 24–27 | 1 each | Active slot / target slot / maximum chunk 42 / reserved zero |
| 28 | 8 | Supported schema |
| 36 | 28 | Active schema/model/revision/size/CRC metadata, or zero if unselected |

Results are 0 OK, 1 malformed/unsupported, 2 stale identity, 3 wrong sequence,
4 not quiescent, 5 store refused, 6 store fault. STATUS success can still report
a faulted store: result and state must both be checked. Resume begins with status
and exact transaction/metadata; it cannot assume an offset or abort unfamiliar work.

## Model control and status

Types 26/27 share the same connection, sequence and retry cache; no second OPEN
is needed. Core operations are:

| Operation | Value | Body |
| --- | ---: | --- |
| STATUS | 0 | Empty |
| STOP | 1 | Expected activation generation u32 |
| SELECT | 2 | Expected generation; cache slot u32; exact 28-byte metadata |
| ACTIVATE | 3 | Expected generation; transaction u32; exact metadata |
| OUTPUT / INPUTS | 4 / 5 | Empty |
| CAPTURE / READ | 6 / 7 | Runtime checkpoint capture / token-and-offset read |
| RESTORE_BEGIN / RESTORE_WRITE | 8 / 9 | Runtime restore staging |
| RF | 10 | Expected generation u32; inhibit 0 or resume 1 |
| RADIO_CAPTURE / RADIO_READ | 11 / 12 | Empty / token u32 and offset u32 |
| HANDSET | 13 | Empty |
| TIMER / TIMER_RESET | 14 / 15 | Timer index u8 / expected generation u32 plus index |
| SOURCES | 16 | Empty |
| SENSOR | 20 | Sensor index u8, 0–59 |
| RESET / RESET_STATUS | 21 / 22 | Expected generation plus reset selector / empty |

The runtime forms and restore-token activation extension are specified in
[runtime transfer](../firmware/g11-model-activation-and-runtime.md#runtime-transfer).
Service availability and configuration schema are separate compatibility checks;
this table is not a claim that every historical peer implements every operation.

Ordinary model status/mutation replies use:

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0–3 | 1 each | Version / operation / result / coordinator state |
| 4 | 8 | Boot nonce |
| 12 / 16 / 20 | 4 each | Connection generation / activation generation / output sequence |
| 24–27 | 1 each | Slot / coordinator error / CRSF flags / output validity |
| 28 / 36 | 8 each | Schema / selected model identity |
| 44 / 48 | 4 each | Model revision / CRC32 |
| 52 / 56 | 4 each | Acquisition / mixing time, MCU microseconds |
| 60 | 1 | Startup flags: pending 1, calibration 2, throttle 4, switches 8, pots 16 |
| 61 | 1 | Bad-switch bits 0–7 |
| 62 | 1 | Low nibble: switch bits 8–11; high: pot bits 0–3 |
| 63 | 1 | Pot bits 4–11; only configured G11 positions valid |

States are Uninitialized 0, Maintenance 1, Starting 2, Running 3, Fault 4.
Slot 2 means no engine model. Identity in Maintenance/Fault does not prove live
control. CRSF flag bits are enabled 1, requested 2, configured 4, yielding normal
combinations 0/4/6/7. “RF” here controls the external channel stream, not Nano
power or measured radio emission. Replayed ACTIVATE may still return its original
Starting reply after the MCU has reached Running; obtain a new STATUS.

## Coherent control observations

OUTPUT and INPUTS replies put state in byte 3's low seven bits and validity in
bit 7. Boot/connection/activation/output identities occupy offsets 4/12/16/20;
acquisition/mixing times are at 24/28. OUTPUT stores sixteen signed int16 channels
at 32, range −1536…1536. INPUTS instead stores thirteen int16 analogs at 32,
range −1024…1024; twelve packed two-bit switches at 58; active stick mode at 61;
and thirteen presence bits at 62. Switch codes are up 0, middle 1, down 2, absent 3.
Absent analogs have both zero value and clear presence.

Values already include MCU calibration/reversal/dead-zone processing. Linux must
not calibrate them again. INPUTS and OUTPUT are separate requests and can observe
different cycles. Invalid observations retain state/generation but zero sequence,
times and data. Valid sequence zero is allowed after wrap. The optional stick-mode
extension requires a compatible UI; an old peer's zero means unknown logical
stick mapping, not mode zero.

An answering UART is not evidence that control is advancing. Linux expires
INPUTS and OUTPUT independently after 200 ms without advancing sequences.
Unsigned forward comparisons allow wrap and reject regression. Observation and
radio-delivery guarantees remain distinct.

## Radio and handset observations

RADIO_CAPTURE freezes a dedicated 128-byte snapshot; READ accepts token plus
offset 0/32/64/96. Each reply has the checkpoint-style slice header and 32 data
bytes. OPEN invalidates the snapshot, not live telemetry. Captures do not overwrite
the runtime checkpoint buffer. A valid transfer can contain no valid RF records.

| Snapshot offset | Bytes | Meaning |
| --- | --- | --- |
| 0–3 | 1 each | Version / valid bits (link 1, battery 2, module 4) / state / zero |
| 4 / 8 / 12 | 4 each | Activation generation / output sequence / capture time |
| 16 / 20 / 24 | 4 each | Link / battery / module ages, µs |
| 28 / 32 | 4 each | Accepted / rejected frame counts |
| 36 | 10 | CRSF link statistics, including signed SNRs |
| 46 / 48 | 2 each | Signed battery voltage/current, 0.1 V / 0.1 A |
| 50 | 4 | Capacity, limited to 24 bits |
| 54 / 55 | 1 each | Battery percent / sensor ID |
| 56 / 60 / 64 | 4 each | Module serial / hardware / firmware IDs |
| 68–71 | 1 each | Parameter count / version / ELRS boolean / zero |
| 72 | 44 | NUL-terminated, zero-padded module name |
| 116 | 12 | Reserved zero |

Valid radio records require Running and enabled CRSF; link age is under one
second and battery/module age under five seconds. Invalid record fields/ages are
zero, with counters still observable. Source age, elapsed host time and advancing
control identity all matter; serial retries cannot refresh a frozen sample.

HANDSET is independent of receiver telemetry. Its reply carries sequence/time/age
at 16/20/24, instantaneous/averaged mV at 28/30, validity at 32, estimated percent
at 33, charger state at 34 and alert at 35, then reserved zeros. Validity bits
are voltage 1, percentage 2, charger 4, alert 8; samples expire at 500 ms. Linux
adds elapsed time since request submission and preserves same-boot sequence
baselines across reconnect. The
[battery study](../soc/g11-battery-and-service-inputs.md) describes conversion,
warning thresholds and measurement limits.

## Timers, sources, sensors and resets

TIMER follows OUTPUT's identity/time header. At 32/36 it carries signed timer
seconds/configured start; bytes 40–43 are state, flags, index, zero; bytes 44–51
are the fixed-width name. Flags are enabled 1, show-elapsed 2, persistent 4.
This is a foreground observation, not historical per-cycle timer storage.
TIMER_RESET requires Running, matching generation and idle calibration/shutdown
gates, marks runtime dirty and does not write flash or change output permission.

SOURCES uses identity through output sequence at 20. A u16 at 24 packs flight
mode in bits 0–3 and nine decimal-precision flags in 4–12; six int16 trims divided
by eight start at 26; nine GVar words start at 38, with signed 12-bit value and
percent-unit bit 12; eight logical-switch bytes start at 56. Linux expands trim
values to int32 source units. It displays already-evaluated MCU inheritance and
switch state without running another evaluation. Timer/source observations expire
after one second without progression and require matching active control.

Optional service bit 4 advertises SENSOR. After the common header through
sequence at 20, signed current/min/max are at 24/28/32; index/flags/unit/precision
at 36–39; four-byte label at 40; remaining lifetime at 44; reserved zeros at 48.
Flags are configured 1, available 2, fresh 4. Expired items retain extrema but
clear fresh/lifetime. A consumer subtracts elapsed time from the **first** request,
including retry time. Linux widgets renew two-second interest leases; the worker
polls at most four slots per pass and no faster than 100 ms per slot.

Optional service bit 8 advertises RESET/RESET_STATUS. Selectors 0–2 reset timers,
3 flight, 4 telemetry, 5 current-mode trims, 6–65 one sensor. Only explicit UI
actions submit resets. RESET_STATUS carries sequence at 20, flight-reset count at
24, startup flags at 28, switch/pot masks at 29/31 and zero padding from 33.
Warnings describe an observation, not RF permission. Exact retry deduplicates
effects; after uncertain acknowledgement, reconnect observes rather than
automatically repeating the reset.

## Evidence and version boundaries

Native framing/store tests transferred all 7,991 bytes, including the odd tail,
and covered lost acknowledgements, stale identities, reconnect, refusal and
unchanged active slots. Actual Linux export was decoded by the ARM validator,
including correct-CRC unsupported-model rejection. Later combined firmware and
physical deployments supersede early “candidate only” paragraphs in the research.
For example, sensor observations were deployed at boot 120; later revisions
added reset behavior without making old schema identities current.

These software/deployment results do not establish real Nano telemetry or receiver
delivery. [Connected-module electrical tests](../experiments/g11-crsf-electrical.md)
still recorded faults and no replies. Persistence and explicit workflow behavior
are covered by [Linux model management](../systems/g11-model-management.md).
