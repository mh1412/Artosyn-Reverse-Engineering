# G11 MCU calibration and Linux save/apply workflow

[Firmware](README.md) · [Input mapping](g11-autonomous-engine.md#confirmed-physical-inputs)

The MCU collector measures raw control travel independently of Linux polling.
It does not change calibration or write flash. Linux saves a complete candidate
and explicitly applies it through the model manager. The existing calibration
continues to control mixing throughout measurement.

## Collection lifecycle

Collection uses mapped 12-bit ADC scans before stick-mode mapping, inversion,
filtering or active calibration. BEGIN binds a nonzero activation generation and
the exact thirteen-bit presence mask from the active MCU configuration. All four
main axes are required; mapped multiposition controls are refused because they
need separate step calibration.

1. Center collection requires main axes and detented controls to remain within
   an eight-unit band in 11-bit calibration coordinates for at least 200 ms and
   sixteen samples. Movement restarts that window. Ordinary undetented knobs
   may use automatic midpoint calculation.
2. Explicit TRAVEL freezes centers and begins extrema collection. Automatically
   centered knobs use the midpoint of their measured extrema.
3. FINISH subtracts `span / 64` from each raw half-span, matching upstream's
   allowance, and requires both spans in 100–2047 for every present control.
   Missing/one-sided travel keeps collection active; no partial result applies.
4. Successful FINISH freezes midpoint/negative-span/positive-span triples.
   Absent entries are zero placeholders, not replacements for saved settings.

Only newly published complete ADC/DMA scans advance collection. Cached reads do
not count. Invalid/missing inputs, skipped acquisition sequences, backwards time
or scan gaps of at least 100 ms fault measurement. Sequence/timer wrap is handled
with bounded intervals. A stale last scan cannot complete center or travel.

BEGIN replaces an earlier measurement; CANCEL discards it. Start requires the
expected Running activation, explicitly inhibited CRSF, no active/faulted radio
transaction and idle shutdown policy. STOP, activation loss, scheduler fault or
power-off request invalidate collection. Canceling shutdown cannot resurrect it.
Policy refusal or collector failure alone does not stop mixing. Linux silence
does not discard a valid measurement; a new management OPEN does.

## Calibration wire contract

Types 30/31 share the
[management connection and exact-retry cache](../protocols/g11-autonomous-management.md#ownership-ordering-and-retry).
Each BEGIN receives a monotonic token which cannot be reused after cancel or
reconnect; exhaustion requires reboot. No operation writes flash.

| Operation | Value | Body after the standard 14-byte prefix |
| --- | ---: | --- |
| STATUS | 0 | Empty |
| BEGIN | 1 | Activation generation u32 |
| TRAVEL / FINISH | 2 / 3 | Activation generation u32; token u32 |
| READ | 4 | Generation u32; token u32; first index u8 |
| CANCEL | 5 | Generation u32; token u32 |

READ accepts first index 0, 3, 6, 9 or 12 and returns up to three entries. Replies
have version/op/result/reserved zero at 0–3, boot nonce at 4 and management
generation at 12. The remaining 48-byte body is:

| Body offset | Bytes | Meaning |
| --- | --- | --- |
| 0–3 | 1 each | Phase / sample-freshness flags / first index / entry count |
| 4 / 8 | 4 each | Measurement token / activation generation |
| 12 / 16 / 20 | 4 each | Acquisition sequence / sample timestamp / phase sample count |
| 24 / 26 | 2 each | Present / automatic-midpoint masks |
| 28 | 6 per entry | Midpoint / negative span / positive span, u16 each |
| Remainder | Variable | Zero padding |

Outside READ, index is 255 with zero count/data. Failed requests have zero bodies.
IDLE STATUS may retain the token counter but no measurement identity. Result
freshness may age after FINISH without changing frozen values.

The worker retrieves all five slices before publishing a complete thirteen-input
result. Every slice must agree on frozen identity, masks and sample count;
partial or inconsistent data never reaches the GUI as a candidate. Expected
center/travel refusals preserve the session for explicit retry. Transport/protocol
failure reconnects and invalidates local results without replaying calibration.

## Save before model activation

Opening the autonomous calibration page does not start measurement. Start and
Continue make explicit transitions; Save and apply confirms a complete result
bound to boot, session, activation and measurement token.

Preparation uses the exact journaled active MCU model, not possibly edited GUI
settings. It updates only mapped ordinary analog calibrations and the upstream
four-main-input checksum, preserving unmapped values. A separate compatibility
check proves that all other model/map/header/radio content remains unchanged.

The GUI also snapshots ordinary `RADIO/radio.yml` settings for a separate save
worker. This replaces only corresponding calibration/checksum fields after
checking mapping/pot compatibility. The write uses a temporary file, checked
file fsync/close, rename and directory fsync. The ordinary EdgeTX writer is
inhibited until the GUI acknowledges the result. Concurrent radio-setting edits
cause failure rather than falsely marking the current state clean.

Only successful acknowledged settings save permits model activation. Save failure
leaves the MCU's current model running and the prepared result available for
explicit retry. Applying the derived candidate from the Controller model page
uses the same gate. Closing during measurement cancels it; closing during queued
save allows that write to finish but vetoes later activation. An already-authorized
activation completes through the model manager.

The derived candidate carries its exact runtime-source identity in the Linux
journal. Apply stops/captures that source and durably reseals its checkpoint under
the new revision/CRC, preserving trims, GVars, sticky state and persistent timers.
Missing state/proof or failed persistence cannot silently select defaults.
Transient filter/fade state and nonpersistent timers are not preserved across
initialization. See [model management](../systems/g11-model-management.md#calibration-derived-state).

## Recorded verification

The collector matched unmodified pinned EdgeTX calibration from commit
`64c7221001332dba962c90abdf76ca9c89f8840b` for 128 cases and 15,872 accepted scans.
Cases include asymmetry, odd readings, automatic/detented controls, variable
presence and counter limits. Fixture static RAM was 232 bytes and observed stack
48 bytes; neither is a complete production memory or worst-case stack bound.

Unicorn 2.1.4's memory-write instrumentation changed execution of a conditional
halfword store, even with a no-op hook, skipping alternate assigned axes. These
tests use Cortex-M4 mode without that hook, while retaining read-only flash,
unmapped-memory traps, stack checks and an SRAM-gap canary. The canary cannot
detect every transient write later restored. Native sanitizers provide separate
memory-safety evidence.

Combined ARM/PTY tests cover independent collection, full-scan counters, exact
retry, frozen reads, reconnect cancellation, policy loss and unchanged storage.
Configuration tests include 31,812 mutations outside allowed fields. The complete
host touch-UI test checked save failure without stopping the MCU, explicit retry,
lost activation reply reconciliation, all six analog values, preserved GVar 42,
cancellation and Linux termination mid-transfer. Restart required explicit resume
through the save gate. Its report explicitly records no hardware access or
deployment: these are simulated ADC/UART/flash/time results, not physical G11
calibration, electrical timing or receiver validation.
