# G11 MCU-owned module configuration and ExpressLRS Lua

[Protocols](README.md) · [CRSF transport](../firmware/g11-crsf-transport.md)

The implemented configuration path is Lua `crossfireTelemetryPush/Pop` → Linux's
sole UART worker → MCU management endpoint → MCU CRSF slot arbiter → module.
Replies return through bounded queues to Lua. The MCU retains channel timing,
mixing, sensor decoding and discovery when Linux exits.

## Admission and ownership

Only admitted discovery, parameter read/write and supported bind/model-select
commands may enter. Channel packets are rejected by Linux and again by the MCU.
Writes/direct commands require fresh enabled output with CH5 ≤ −512. This is
the configured legacy CH5 low state, not proof that a receiver is disarmed.
The arbiter rechecks immediately before sending and discards a queued write if
CH5 rises. Reads/discovery may run with CH5 high.

Management uses at most alternate slots and never replaces the first channel
frame. At 4 ms slots this can produce 8 ms channel spacing. The endpoint cannot
enable output, stop a model, write flash or wait synchronously for a module reply.
Calibration, shutdown, inhibition, fault and activation changes invalidate its
transfers; explicit inhibit does so even if inhibit/resume occurs between polls.

## UART endpoint

Model SERVICES operation 17 returns the capability bitmap at reply byte 16;
bit 0 advertises module configuration. Bytes after 19 are zero. Unsupported
peers are reported as such without repeated probing/reconnect. Requests/replies
use types 32/33 and the existing boot/session/management-generation/sequence cache.

The standard 14-byte request prefix precedes these little-endian bodies.
Generation here is the model activation generation:

| Operation | Value | Body |
| --- | ---: | --- |
| STATUS | 0 | Empty |
| BEGIN | 1 | Generation u32; command length u8, 6–64 |
| WRITE | 2 | Generation u32; token u32; offset u8; 1–41 bytes |
| COMMIT | 3 | Generation u32; token u32 |
| CAPTURE | 4 | Generation u32 |
| READ | 5 | Generation u32; RX token u32; offset u8, 0/24/48 |
| CLEAR | 6 | Generation u32 |

Incomplete commands expire after 250 ms. Sequential WRITE retransmission with
matching bytes is idempotent. COMMIT of an already-admitted token returns OK
without another submission. Arbiter command age is bounded to 100 ms. Thus OK
means admitted, not sent or acknowledged: expiry, inhibition or changed CH5 can
discard a command before its slot.

After the common sixteen-byte reply prefix, the 48-byte body is:

| Body offset | Meaning |
| --- | --- |
| 0 | Flags: ready 1, low-CH5 write permission 2 |
| 1 / 2 / 3 | TX state (idle 0, collecting 1, admitted 2) / length / received count |
| 4 / 8 | Activation generation u32 / TX token u32 |
| 12 / 16 | Frozen RX token u32 / MCU receive timestamp u32 |
| 20 / 21 / 22 / 23 | Frame length / slice offset / slice length / queued replies 0–2 |
| 24–47 | Up to 24 frozen bytes, zero-padded |

Only CAPTURE/READ carry RX metadata; empty CAPTURE zeros it, and refusal zeros
the entire body. Linux validates framing, CRC, length, tokens, identities,
timestamp continuity, addresses and reply type before publishing a complete frame.

Configuration replies `0x29/0x2b/0x2d/0x2e/0x32` may enter from TX/receiver
`ee/ec` to `ea/ef` or broadcast. ELRS `0x2e` requires bad/good packet counts,
flags and a terminated warning string. Sensors and timing correction stay on
the MCU path. The two-entry MCU queue drops new arrivals when full; queued/frozen
replies expire at 500 ms. Linux's four-entry reply queue also ages observations.

OPEN clears partial upload and reply snapshots without replaying or revoking an
already-admitted arbiter command. Exact packet retries use the common cache.
If COMMIT acknowledgement remains uncertain, reconnect does not retry the entire
module command. Lua queue acceptance cannot be presented as module acknowledgement.

## Lua worker and screen behavior

Availability uses actual MCU services and applied output state, not the editable
Linux draft's protocol. A four-command FIFO permits one active and three waiting
commands for script bursts. The worker interleaves replies and control observations;
model/power jobs remain exclusive. Refusal, disconnect or uncertainty discards
waiting commands rather than replaying them later.

Replies enter upstream Lua queues directly. This path does not invoke upstream
device-info handling that would automatically edit the local model or apply
Linux timing corrections. Lua observations clear on disconnect, inhibit or
boot/connection/model changes. Parameter chunk interpretation and command polling
remain the script's job.

Tests ran unmodified hash-pinned ExpressLRS 3.3.0 and Lua r18, the latter pinned
to commit `c051c3dc6cd4bca49c26eaa4205d5d264c4defb9`. The older script overlapped
text at 1920×1080; r18 computed layout from display/font metrics and produced
separate readable rows. That finding does not establish compatibility with the
unidentified physical Nano firmware.

The deployed r18 script SHA-256 was
`c52bed26ff02f84e5857f01b8c88dbac95029e8f46e6e6cef398419ad91f0e14`.
An RF-unconfigured instruction screen opened on the actual G11. Initial testing
used an injected long EXIT; the screen lacked a touch exit. A later UI-only
native **Close tool** overlay supplied it, survived script clearing/errors in
host tests, and passed physical injected-touch close/reopen without restarting
the UI during that interaction. The UI replacement itself changed PID.

## What was verified

Endpoint/client tests cover boundaries, malformed traffic, exact retry, lost
commit ACK, expiry/wrap, queue overflow and CH5 rising before transmission.
Actual native UI/worker plus combined ARM tests performed discovery, multi-chunk
parameters, a 64-byte write, status, edit/readback and rejected channel injection.
Module replies and peripheral time were simulated; Linux mixing remained zero.

Boot-110 deployment used MCU
`ceedf2667b67b6e413ec9609143c875fea08b88149ba9f728542b7f49199ffc8`.
The selected model, working overlay and options were preserved, Linux/UI returned,
and RF-off mixing was 499.79–500.19 Hz. Its memory footprint was 38,988 bytes fast
code, 18,628 stopped code, 11,536 static RAM and a 2,800-byte gap plus reserved
2,048-byte stack. Observed ARM-test stack 1,272 bytes is not a worst-case bound.

These are deployment, UI and simulated-module results. Later
[physical electrical tests](../experiments/g11-crsf-electrical.md) located output
on RX/PPM and still failed to establish connected Nano replies. Early wiring
assumptions in module notes do not supersede that later evidence. Real parameter
readback, half-duplex timing and receiver output remain unqualified.
