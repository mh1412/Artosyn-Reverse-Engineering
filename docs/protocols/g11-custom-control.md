# G11 custom Linux-mixer control protocol

[Protocols](README.md) · [G11](../devices/skydroid-g11.md)

## Version and architecture scope

This is the custom protocol used by the historical Linux-mixer/controller
design. It must not be sent to stock firmware. Its Linux-produced outputs and
input deadlines are also distinct from the later autonomous MCU, where Linux
provides management/UI and the MCU owns mixing. Shared framing alone does not
make those message sets interchangeable.

The original protocol notes contain early unvalidated-board and no-install
statements. Later hardware records supersede those blanket statements for the
specific UART and recovery tests documented in
[UART/session lifecycle](../firmware/g11-uart-and-session-lifecycle.md).
They do not establish successful external receiver delivery.

## Envelope

Integers are little-endian. The decoded packet is:

| Offset | Bytes | Field |
| --- | --- | --- |
| 0 | 1 | Version 1 |
| 1 | 1 | Message type |
| 2 | 4 | Session token |
| 6 | 4 | Input sequence or independent observation sequence |
| 10 | 4 | MCU acquisition/reassembly timestamp, µs |
| 14 | 0–64 | Payload |
| End−2 | 2 | CRC16-CCITT, initial `0xffff`, polynomial `0x1021` |

CRC covers header and payload. COBS encodes the whole packet, followed by a
zero delimiter. UART length is payload+18 bytes, maximum 82. Invalid frames
are abandoned at the delimiter. CRC detects corruption but does not authenticate.
Control messages use zero sequence/timestamp; data retains original MCU
identity. Linux must not invent a fresh acquisition time or directly compare
MCU timestamps with its own clock.

## Discovery and session identity

| Type | Name | Payload |
| --- | --- | --- |
| 5 | DISCOVER | Empty; token zero |
| 6 | OFFER | boot nonce:u64, generation:u32, capabilities:u32, link baud:u32; token zero |
| 7 | OPEN | nonce:u64, generation:u32, sample period:u32, RC period:u32, timeout:u32, link baud:u32 |
| 8 | OPENED | Exact OPEN payload |
| 9 / 10 | CLOSE / CLOSED | nonce:u64, generation:u32 |
| 12 / 13 | ENTER_MAINTENANCE / ACK | nonce:u64, generation:u32, nonzero request ID:u32 |

OPEN uses a fresh nonzero host token. Discovery is observational. An identical
OPEN retry gets an ACK without resetting sample history, deadlines or outputs;
a different host cannot preempt a live session. Close or output expiry disables
RC and advances the generation. Exhaustion fails shut. An opening host may
close to cancel an OPEN whose ACK was lost. Host recovery is explicit with a
new token, not automatic retry of the expired session.

The board must provide a nonzero 64-bit boot nonce not reused across resets.
A fixed chip UID or restarting timer cannot provide this. The persistent
allocator and its reserved storage are part of the board implementation; early
simulation evidence alone was not physical reset protection.

| Capability bit | Meaning |
| --- | --- |
| 0 / 1 | Raw inputs / output with explicit RF enable; both required by host |
| 2 | Historical native-update transition; not advertised by later custom runtime |
| 3 / 4 | Inbound CRSF telemetry / outbound management frames |
| 5 | Freshness-bound encoded CRSF output; required by configured native CRSF UI |
| 6 | Resident maintenance transition with stop/drain/shutdown/handoff support |
| 7 | Board voltage status |
| 8 | Historical coordinated power-off |
| 9 / 10 | Feedback / independently advertised auxiliary actuator support |

## Deadlines and line budget

Negotiated input and RC periods are 2–20 ms; timeout is 10–250 ms and must
exceed both periods. The board additionally rejects RC periods below its
transport floor, 3,525 µs for the described 400 kbaud maximum command/reply
schedule. OFFER does not advertise that floor; unsupported OPEN gets no ACK.
Negotiation validates an already configured baud; it does not change UART rate.

The host bounds discovery/OPEN to one second including retries. OPENED starts
its first-input deadline; only valid newer raw input renews reception freshness.
The MCU starts its independent first-output deadline when accepting OPEN and
subsequently ages results from their original acquisition timestamp. Duplicates,
malformed traffic and repeated ACKs cannot keep either side alive.

Raw input occupies 68 UART bytes and normalized output 51. Their conservative
sequential 8N1 budget is 10.33 ms at 115200 or 1.29 ms at 921600. Thus 5/10 ms
input periods fail this validator at 115200 while 20 ms fits. Processing,
queues, telemetry and scheduling require additional margin. Physical later
921600 tests do not turn this arithmetic into a latency guarantee.

## Raw samples and accepted outputs

Type 4 RAW_INPUT has 50 payload bytes:

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0 | 32 | Sixteen 12-bit ADC readings stored as u16, indexed by ADC channel |
| 32 | 2 | ADC-present mask |
| 34 | 8 | Four u16 valid-pin masks, ports A/B/C/F |
| 42 | 8 | Four u16 high-level masks in the same order |

Absent ADC values are zero; high GPIO bits must be valid. Malformed decoding
leaves the destination unchanged. These are electrical observations requiring
board mapping and calibration, not already ordered sticks or debounced controls.

Type 11 CONTROL_OUTPUT echoes token/sequence/timestamp and carries sixteen
signed i16 channels followed by one RF-enable byte. Channels span −1536…1536
(nominal ±100% is ±1024); enable is only 0 or 1. Conversion uses CRSF center
992, 4/5 scaling and saturation to 0…1984. The session rejects legacy bench
type 2 OUTPUT. Fresh disabled results keep input observation alive but stop RC.

The host accepts a mixer response only for its latest complete snapshot and
discards it if a newer sample arrived during mixing. The MCU checks a 16-entry
issued-sample history and rejects reordered/expired results. Acquisition alone
does not refresh an old mixed result.

Type 15 CRSF_OUTPUT carries one enable byte and a complete 26- or 27-byte
channel frame. Validation requires address `0xee`, type `0x16`, correct
length/CRC, sixteen 11-bit values in 0…1984, and optional arming status 0, 1 or
2. Status 0/1 is switch-disarmed/armed; 2 selects CH5 arming. Reserved status
bits are rejected. It is not arbitrary UART transmission.

The native UI used the pinned EdgeTX channel-frame builder to retain center
adjustment, saturation and arming mode. Both output forms consume a sample at
most once and share freshness checks. Only the latest accepted result is kept;
old RC frames are not queued. Disabled output clears pending management. The
27-byte channel variant occupies 46 bytes on the internal UART, within the
normalized-output budget. The external period floor still budgets a maximum
management frame and reply.

The pinned decoder falls back to CH5 arming for reported ELRS versions below
4.0. Synthetic 3.3.0/4.0 peers exercised that behavior; the installed Nano
version and physical behavior were not established.

## Telemetry and module management

Type 3 TELEMETRY carries one complete CRC-valid 4–64 byte CRSF reply, using
the live token, independent increasing sequence and MCU reassembly-start time.
It never renews control deadlines. The MCU reads at most 64 external bytes per
poll; a 50 ms inter-byte gap or 100 ms total assembly age discards a partial
frame. Replies can span external receive opportunities. Four complete frames
may queue; overflow is counted, and frames older than 100 ms expire.

Input/RC has priority. Telemetry uses an idle internal UART only when its wire
time plus 100 µs fits before the next sample. Linux validates token, sequence,
CRSF address/CRC and age within 100 ms of its latest MCU input timestamp, then
uses a separate four-frame queue with 100 ms host residence. Session loss clears
telemetry; channel packets cannot inject trainer input through this path.

The session core itself has one pending telemetry slot. A newly accepted reply
replaces an unconsumed reply in that slot; the outer host client must drain it
into its four-frame queue. That queue does not turn the core API into a
four-frame buffer. Telemetry requires an increasing sequence but may reuse a
timestamp within the permitted age window. Raw input and power updates require
both sequence and timestamp to advance.

Type 14 MODULE_COMMAND carries a 6–64 byte CRSF management frame tied to a
freshly answered RF-enabled sample. The MCU requires the latest accepted result
identity and a newer command sequence, even if the command is dropped because
its one pending slot is full. Commands never renew control freshness.

Allowed commands are ping `0x28`, parameter read/write `0x2c/0x2d`, and bind/
model-selection forms of `0x32`, with both command CRCs checked. Origins `0xea`
(radio) and `0xef` (stock ELRS Lua) are accepted. Channel injection, baud-change
negotiation and other classes are rejected. Linux queues at most four commands
for 100 ms and appends at most one after a mixer response if the combined wire
budget plus 100 µs fits. MCU pending commands expire 100 ms after their sample.

The scheduler sends channels first and lets management replace at most every
other external slot, never consecutive management slots. Sustained management
can therefore double the channel interval before turnaround delays. The internal
receive budget is 256 bytes/poll. No new transport-level command ACK or retry
is added; requesting scripts use CRSF replies. RF disable, close, expiry and
maintenance discard pending commands.

## Voltage and historical power messages

Type 16 has four LE16 fields: battery ADC, reference ADC, estimated VDDA mV and
battery mV. It has its own sequence and original acquisition timestamp, carries
no charging/percentage field and never renews control freshness. Emission is
at most once per 500 ms, after input/telemetry, when its 26-byte UART frame plus
100 µs fits. Post-OPEN acquisition must be less than 100 ms old; the GD32
acquisition producer has a tighter 2 ms freshness rule.

Linux requires a live session, newer identity, age within 100 ms of latest raw
input, 12-bit ADC values, nonzero reference, VDDA 2600–3600 mV and battery
2000–5000 mV. Data expires after two host seconds or immediately on session
loss. These are range and freshness checks: the protocol validator does not
recompute VDDA or battery voltage from the transmitted ADC counts. Estimation
belongs to the acquisition producer, so accepted fields need not be mutually
consistent merely because they passed wire validation. Charging comes
separately from raw PB11/PF7; see
[battery inputs](../soc/g11-battery-and-service-inputs.md).

Historical types 17–22 are POWER_PREPARE/PREPARED, COMMIT/COMMITTED and
CANCEL/CANCELLED. All use zero sequence/timestamp and a 16-byte nonce,
generation and request-ID payload. PREPARE consumes the live generation,
stops RF and reserves 30 seconds; exact retries do not extend it. COMMIT must
match, and hardware TC of its reply precedes the off callback. Timeout/cancel
leaves power on and requires a fresh control session. A lost COMMIT ACK is
ambiguous. This is superseded for the autonomous board by
[types 28/29 and physical-request authorization](../firmware/g11-power-and-shutdown.md#autonomous-shutdown-messages).

## Maintenance and implementation limits

Maintenance validates epoch, request ID and ownership, disables RF and retires
the session before queuing its ACK. The described runtime requires actual
hardware TC within 100 ms before shutting down producers and handing off;
late TC faults rather than authorizing a delayed transition. The host accepts
only the exact ACK within one second and releases its UART on success or
failed maintenance. ACK does not prove the loader started: a new 115200
HELLO/OPEN exchange is required. Later direct handoff superseded early
assumptions that every maintenance transition must physically reset the MCU.

The session implementation has no heap and one foreground owner per endpoint.
Compared 32-bit clock intervals must stay below `2^31` µs; discontinuity or
long suspension requires reinitialization. Driver faults terminate producers
and watchdog feeding. The MCU's independent watchdog and board power behavior
remain separate from protocol correctness. Host/PTy/ARM tests cover retries,
epochs, malformed frames, timing and real mixer integration; physical receiver
behavior remains outside those tests.

A November 2 host recheck of the retained session tests passed with address and
undefined-behavior sanitizers enabled. It exercised raw-value rejection,
negotiation/replay/loss, RF cutoff, encoded CRSF validation, power and feedback
messages, maintenance deadlines and clock wrap. This was an offline check of
the historical Linux-mixer session; it adds no physical UART, ADC, RF or
receiver qualification.
