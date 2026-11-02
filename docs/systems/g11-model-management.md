# G11 Linux model preparation and durable apply

[Systems](README.md) · [G11](../devices/skydroid-g11.md) · [MCU persistence](../firmware/g11-model-activation-and-runtime.md)

Linux edits and stores the model library; the MCU owns the active control model.
Opening the UI, editing settings or selecting a local model does not apply it to
the MCU. The Controller model page provides separate Prepare and Apply actions.

## Prepare and Apply

Prepare copies the selected model, radio settings and verified physical map under
the GUI mixer mutex. A worker validates/encodes and durably saves the candidate
outside that mutex. It sends no UART command and does not stop control.
Unsupported settings are reported without silently rewriting the model.

Apply confirmation binds the exact candidate ticket, MCU boot nonce and activation
generation. Before any mutating UART request the worker saves the full pending
candidate, then performs this sequence:

1. Stop a different running/starting model, subject to the MCU's local permission
   and CRSF-inhibition requirements.
2. Capture the initialized outgoing model's frozen runtime and save it durably.
   Failure leaves control stopped and the pending candidate available for retry.
3. Transfer the candidate to READY, or select an exactly matching cached model.
4. Upload the incoming model's saved runtime if present, activate, and wait for
   matching Running identity plus fresh output. Starting alone is insufficient.
5. Save Linux's confirmed active record before removing pending.

An already-running exact match only updates the observation; it does not reset
the engine. The sole UART worker rechecks boot identity immediately before a job,
including after reconnect. Model filesystem work has its own worker and lock.

## Identity and file records

The on-device `.g11-controller` directory holds one latest candidate per filename
in `models/`, exact-identity runtime records in `runtime/`, a full `pending`
candidate and the last observed `active` record.

A filename retains a nonzero random 64-bit model identity. Revision begins at 1
and advances when encoded content changes; identical content keeps its revision
and nonzero 32-bit transaction token. Exhaustion refuses further increments.
Rename/copy to another filename currently creates another identity. These tokens
provide consistency rather than authentication.

The ordinary little-endian file has a 112-byte header:

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0 | 8 | `G11MODL1` |
| 8 / 16 | 8 each | Model identity / schema |
| 24 / 28 / 32 / 36 | 4 each | Revision / transaction / payload size / payload CRC32 |
| 40 | 1 | Filename length, 1–63 |
| 41 | 3 | Reserved zero |
| 44 | 64 | Filename with zero padding |
| 108 | 4 | CRC32 of bytes 0–107 |
| 112 | Payload size | Exact configuration blob |

Writes use exclusive temporary creation, complete write, file fsync, atomic
rename and directory fsync. Creating directories also synchronizes their parent.
Validated names, no-follow opens and regular-file checks constrain access.
Corrupt or incompatible records are errors, not permission to invent new IDs.
Incomplete unnamed temporary files cannot replace committed records.

Pending preserves its own payload independently of later edits. Another unresolved
pending transaction cannot be overwritten. An exact retry re-synchronizes the
record, including after a rename followed by a reported directory-fsync failure.
Saving active before deleting pending preserves enough identity for reconciliation.

## Calibration-derived state

Calibration changes configuration revision/CRC, but should preserve live trims,
GVars and persistent timers. The narrowly defined migration first proves that
model/map/non-calibration radio bytes are unchanged and only mapped analog
calibration plus its checksum differ. Matching identities alone do not prove this.

`G11MODL2` uses a 144-byte header. Bytes 8–107 retain the ordinary fields, followed
by source schema at 108 (u64), source model at 116 (u64), source revision/size/CRC
at 124/128/132 (u32), reserved zero at 136, header CRC over 0–139 at 140, and
payload at 144. Source and candidate share model/schema/size, with an older source
revision. The runtime-source requirement survives Linux exit.

If the source remains active, Apply stops/captures it and durably saves its exact
checkpoint. Migration changes only checkpoint revision/configuration CRC and
reseals it, preserving capture metadata and runtime values. The derived checkpoint
must be saved before candidate transfer. Missing required state or proof fails;
there is no fallback to default runtime for calibration-derived candidates.
Ordinary models without a saved checkpoint may use configuration defaults, while
corrupt saved checkpoints always fail.

## Interruption and reconciliation

A failed/uncertain UART acknowledgement ends the apply sequence. Reconnect does
not automatically send later mutations. If fresh status proves the pending
candidate already Running, the worker records that observation without issuing
control commands. Filesystem reconciliation errors are reported rather than
retried continuously in the background.

After restart, unresolved pending work requires explicit Apply or Discard. Apply
reuses exact identity/payload and the MCU's reported state. Discard requires a
matching acknowledged ABORT or idle store with no transaction; an unrelated MCU
transaction is not aborted. Neither Discard nor reconnect stops or activates
control. Linux's active file is an observation, never a replacement for the
MCU's boot-selection journal.

## Evidence boundary

Native sanitizer and process-interruption tests cover identity, file ordering,
locking, reported fsync failure and interrupted commits. PTY tests exercise
journal-before-UART ordering, stale confirmation, lost activation replies and
explicit resume/discard. Actual native touch UI plus combined ARM execution
tested matching payloads, mid-transfer Linux termination and restored GVars.
The same simulated MCU/flash survived replacement of the PTY after Linux exit.
These tests do not establish physical UART recovery or power-loss durability on
the G11 filesystem. Physical MCU reset evidence is recorded separately in
[activation and runtime](../firmware/g11-model-activation-and-runtime.md#qualification).
