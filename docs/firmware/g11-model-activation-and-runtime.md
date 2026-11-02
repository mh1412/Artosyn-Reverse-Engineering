# G11 model activation, selection and persistent runtime

[Firmware](README.md) · [Autonomous engine](g11-autonomous-engine.md)

A durable candidate, a selected model, a running mixer and enabled CRSF output
are different states. The MCU owns those transitions. Linux manages a model
library and requests explicit changes through the fenced management connection.

## Activation and output permission

| Coordinator state | Acquisition/control | Storage work |
| --- | --- | --- |
| Maintenance | Stopped | Allowed only through the additional local ownership gate |
| Starting | Acquisition/calibration starting | Refused |
| Running | Live control | Refused |
| Fault | Stopped, terminal fault latched | Refused |

SELECT requires an idle store and exact cached identity; ACTIVATE requires the
exact READY transaction. Both require the expected activation generation and
refuse while running, before accessing an alternate model. Ordinary upload,
OPEN and reconnect do not implicitly stop control. STOP requires local policy
permission and first requires explicit inhibition if channel output is enabled.

Activation validates CRC, metadata and model semantics, initializes the engine,
optionally restores runtime, and durably commits selection **before** creating
the ADC owner. Persistence failure faults activation without starting inputs.
A successful acceptance advances activation generation and enters Starting.
Fresh acquisition must become ready within the startup allowance, and only a
successful real mix establishes Running. The accepted acknowledgement is not
proof of either Running or radio output.

Startup checks use the MCU's completed input scan, saved calibration checksum
and configured throttle/switch/knob warnings. Models with time dependencies also
require a valid [MCU clock](../systems/g11-clocks.md#time-as-an-mcu-model-source). They wait for a successful
10 ms function tick so an override can take effect before output. A configured
model requests output after activation unless a same-boot inhibit is latched.
Clear checks do not define vehicle arming or prove over-the-air transmission.

Explicit inhibit aborts/releases serial output while acquisition/mixing continue.
It survives same-boot model changes, calibration and Linux reconnect/restart.
Explicit resume or MCU reset clears the latch. Cold boot may restore a selected
model and start output without Linux. Startup warnings are not continuous
cutoffs after transmission begins. A physical shutdown request similarly leaves
live control intact until committed/forced shutdown. See
[power sequencing](g11-power-and-shutdown.md).

## Selected-model journal

Two 1 KiB pages at `0x0800d000` and `0x0800d400` contain authoritative selection.
The newer staged cache and the disposable active mirror cannot override it.
Boot revalidates the selected cache and restores its corresponding runtime before
starting acquisition; missing, corrupt or mismatched records do not trigger a
guess at another slot. STOP does not erase selection, so reset can restore it.

The legacy 64-byte little-endian record is:

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0 | 4 | `G11S` |
| 4 / 6 | 2 each | Version 1 / length 64 |
| 8 | 8 | Nonzero, nonwrapping selection sequence |
| 16 / 20 | 4 each | Slot 0 or 1 / reserved zero |
| 24 / 32 | 8 each | Schema / model identity |
| 40 / 44 / 48 | 4 each | Revision / payload size / payload CRC32 |
| 52 | 4 | Reserved zero |
| 56 | 4 | CRC32 of bytes 0–55 |
| 60 / 62 | 2 each | Final markers `a55a`, `5aa5` |

Version 2 is 416 bytes. It retains identity fields through byte 51, sets the
runtime-present word at 52 to 1, stores the 352-byte checkpoint at 56, places
CRC32 of bytes 0–407 at 408 and final markers at 412/414. Both inner and outer
integrity/identity must pass. Simultaneous legacy and v2 markers are corrupt.
Legacy restoration uses configuration defaults and migrates to v2 before inputs
start.

An update erases the alternate page, verifies all 1,024 bytes, programs/readbacks
the record and writes markers last. The current page remains untouched. Both
valid pages select the larger sequence; equal-sequence disagreement is corrupt.
**A committed-but-invalid record on either page refuses startup even when the
other page is valid.** A torn erase can preserve an old marker and cause this
deliberate refusal. There is no automatic repair or current UI repair operation.

After uncertain mutation, in-memory journal state must be reloaded; an FMC fault
can additionally require reset. Repeating the same persistent values is wear-free:
capture timestamps, generations and dirty counters alone do not cause rewriting.
Normal/forced shutdown stops producers, captures final runtime and commits it.
Failure does not prevent deliberate shutdown; the old page remains, subject to
the same corruption-refusal rules. Abrupt loss restores the last completed
checkpoint, not later unsaved RAM. There are no periodic live-control flash writes.

## Runtime checkpoint format

The 352-byte checkpoint is explicitly serialized little-endian, independent of
compiler bitfield layout:

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0 | 8 | `G11RUN01` |
| 8 / 16 | 8 each | Schema / model identity |
| 24 / 28 | 4 each | Revision / configuration CRC32 |
| 32 / 36 / 40 / 44 | 4 each | Activation generation / output sequence / acquisition time / dirty counter |
| 48 | 162 | 81 signed 16-bit GVars: nine flight modes × nine |
| 210 | 108 | 54 signed 16-bit trims: nine modes × six |
| 318 | 8 | Persistent sticky-switch state |
| 326 | 12 | Three signed 32-bit persistent timer values |
| 338 | 1 | Persistent-timer bitmap |
| 339 | 9 | Reserved zero |
| 348 | 4 | CRC32 of bytes 0–347 |

Schema/model/revision/configuration CRC must match exactly. Activation generation
is nonzero; output sequence zero is valid after wrap. Nonpersistent timers/sticky
values must be zero. Inheritance modes remain configuration, and transient fade,
filter and fractional timer state is not stored.

Capture runs between mixes without flash writes or stopping control, or from a
frozen initialized engine in Maintenance. Starting, Fault and uninitialized
engines refuse. Restore is permitted once after initialization and before the
first mix. It validates the complete record and value semantics before mutation;
invalid runtime cannot silently fall back to defaults.

## Runtime transfer

Model operations share types 26/27 with the
[management protocol](../protocols/g11-autonomous-management.md). CAPTURE (6)
freezes one checkpoint; READ (7) supplies a token and 32-byte-aligned offset
0–320. Its 64-byte reply carries version/op/result/valid at 0–3, boot nonce at 4,
management generation at 12, token at 16, activation generation at 20, output
sequence at 24, total/offset as u16 at 28/30, and a 32-byte slice at 32.
Every slice must match identities/token and the assembled CRC.

Exact retry returns the frozen reply, not a new sample. OPEN/new capture
invalidates it; token exhaustion refuses instead of wrapping. A failed callback
invalidates the capture. A new restore upload reuses this runtime buffer.

RESTORE_BEGIN (8) supplies expected activation generation plus 28-byte model
metadata. RESTORE_WRITE (9) supplies token, offset and 1–42 contiguous bytes.
All 352 bytes must arrive in Maintenance under the same generation/gate. Replies
report token at 16, received count at 20, expected generation at 24 and total
length at 28. Staging itself does not write flash.

A restore-bearing SELECT/ACTIVATE appends the token to its usual 36-byte body.
While restore is staged, ordinary activation without that token is refused.
Complete identity/CRC/generation checks precede coordinator entry; accepted
activation consumes the restore. OPEN invalidates the upload, preventing replay
in a new connection. The activation journal then persists the initialized/restored
values before ADC start.

## Qualification

Simulated journal tests covered 396 initial legacy mutation cuts, 2,508 v2 cuts,
836 migration cuts and every pre-marker byte's corruption. Actual ARM/FMC-model
tests added 99 legacy and 627 runtime interruption cases. Combined tests restored
runtime before first output, checked shutdown save failures and trapped upper
flash execution/data reads while acquisition was active. These do not qualify
physical interrupted programming, flash stalls or power transitions.

On October 25 one physical SWD system reset with the Linux UI stopped advanced
boot 95 → 96 and restored the RF-off bench model. The first post-reset sample
was Running; output progressed at 500.124 Hz over 4.50 seconds. Cache, selection,
options and all other flash outside the nonce region were unchanged. This is
one model/reset, not all runtime permutations or a physical power-cut test.

Debugger restore value 1 means restored/Starting, not necessarily Running.
Final-checkpoint values distinguish not attempted (0), saved/unchanged (1) and
failed (2). Neither diagnostic alone establishes receiver continuity.
