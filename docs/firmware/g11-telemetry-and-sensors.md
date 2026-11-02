# G11 MCU telemetry, model sensors and numeric transforms

[Firmware](README.md) · [Management observations](../protocols/g11-autonomous-management.md#radio-and-handset-observations)

The MCU decodes radio records and evaluates model sensors independently of Linux.
Linux displays MCU values and edits future configuration. Later revision-14
numeric transforms supersede early notes that reject all transformed sensors;
actual physical Nano telemetry remains unverified.

## Decoded records and freshness

| CRSF type | Retained information | Local expiry policy |
| --- | --- | --- |
| `0x14` | Link statistics | One second |
| `0x08` | Receiver battery voltage/current/capacity/percent and optional ID | Five seconds |
| `0x29` | Transmitter identity, versions, parameter count and ELRS signature | Five seconds |

RSSI remains unsigned negative-dBm magnitude in the raw record; SNR is signed.
Battery voltage/current retain signed 0.1 V/0.1 A units and capacity all 24 bits.
An absent optional battery ID means zero. Only the latest battery instance is
stored; different instances are not silently combined. RF profile and raw power
remain module enums rather than inferred physical measurements.

Device information needs a bounded NUL-terminated name and complete fourteen-byte
suffix. Only TX-origin replies addressed to handset/broadcast replace transmitter
identity. CRC, direction, length, percentage, age and timestamp ordering are
checked before updating state. Invalid/unsupported frames increment diagnostics
without replacing accepted data. Raw-frame reception does not establish support
for every sensor or timing message.

Expiry runs without Linux and clears validity before counter wrap could make old
data appear fresh again. Diagnostic values may remain in RAM. Link quality zero
is a received report of a disconnected receiver; battery or module identity alone
cannot prove receiver delivery. Inhibition, model activation and terminal faults
reset telemetry; Linux reconnect does not. Read-only frozen snapshots are separate
from live sensor state and runtime checkpoints.

## Autonomous discovery

With enabled CRSF and fresh control, the MCU sends directed device pings `0x28`
to TX `0xee` from handset `0xea`. Missing identity allows at most one per second;
valid identity suppresses pings until age four seconds, before five-second expiry.
Unanswered discovery does not stop channel output or enable output on its own.

The existing single management mailbox and slot arbiter own transmission. Busy
mailbox contents survive; the first frame is channel data, and management cannot
occupy consecutive slots. At 4 ms slot cadence a ping can extend a channel interval
to 8 ms. Those are scheduling rules, not physically qualified reply-window timing.
Simulated-module tests cover retry, wrong origin, refresh, Linux loss and lifecycle
reset without flash changes.

## Mapping into EdgeTX model sensors

Before each mix the engine projects records into its allocated `TelemetryItem`
array using immutable active model definitions:

| Sensor family | Model ID | Instances |
| --- | --- | --- |
| Link | `0x14` | 0–9 in CRSF link-field order |
| Battery | `0x08 \| (sensor_id << 8)` | 0 voltage, 1 current, 2 capacity, 3 percent |

Sub-ID is zero. RSSI becomes negative dBm, including magnitudes above 127. Power
enums 0–8 use the pinned CRSF mW table; unknown values map to zero. ELRS identity
selects one-based antenna numbering and is retained until radio lifecycle reset.
Numeric setters preserve upstream extrema behavior, including a new voltage
maximum resetting its minimum. Expiry retains values/extrema while marking them
old; activation clears the model items.

Logical conditions can use current/min/max and inverted sensor sources. Presence
requires a received unexpired item: `!isOld()` alone would admit never-received
data. Streaming requires fresh link data with nonzero quality. Single-source
threshold conditions additionally require streaming, while two-source comparisons
require each source individually. Upstream duration/delay state still applies;
an active duration may finish after its underlying condition becomes false.

Scaled Input lines admit current/min/max and inverted sources. Scale 1–16383 is
the positive sensor value corresponding to +100%; for native battery units,
200 means 20.0 V. Scale zero uses upstream unscaled/clamped behavior. Missing
or expired data clears the line's value/trim association without claiming it,
allowing a later eligible fallback. No fallback yields zero.

Saturating before scaling multiplication preserves the eventual ±1024 clamp and
avoids overflow for full 24-bit capacity. Ordinary curves, weights, offsets and
flight modes then run in the existing mixer. Direct unscaled telemetry operands
in mix/weight/GVar positions, undefined sensors and scale on non-telemetry inputs
remain outside this admitted path.

## Numeric transforms in revision 14

Revision 14 implements pinned numeric ratio, signed offset, precision 0–2,
positive-only, auto-zero and four-sample average behavior. Current/extrema,
filter history and auto-zero reference live on the MCU. Scaled Inputs, logical
conditions and Linux widgets consume those same transformed values.

For these raw CRSF units, changing numeric destination unit changes label and
precision; there is no matching upstream physical unit-conversion entry. RPM
uses its separate `raw * offset / ratio` branch and bypasses positive-only,
auto-zero and filtering. Calculated/persistent sensors, sub-IDs, precision 3 and
nonnumeric units remain unsupported.

Admission proves the full decoded range safe, including signed 16-bit voltage/
current and 24-bit capacity, auto-zero differences, filter sums and logical
threshold subtraction. Capacity ratio 128 fits its range; 129 does not.
Validation uses wider intermediates but narrows proven bounds before division.

Only a new received record advances filtering. Polling/retrying observations
cannot filter again. ELRS identity may renumber a live antenna once; repeated
identity reports cannot repeatedly advance its filter. Expiry preserves history;
explicit sensor/flight reset clears it and requires new data.

## Reset and independent receipt counters

Revision 12 added all-telemetry or single-slot reset in model/global functions,
evaluated at the existing 10 ms boundary. A held reset intentionally clears on
every active function tick. Reset clears item values, extrema and receive
validity; whole-telemetry reset also clears streaming.

Link/battery receipt counters distinguish new packets even at equal timestamps
or counter wrap. Reset preserves consumed cursors, so releasing the switch cannot
resurrect cached data. Fresh battery cannot restore link/streaming, and fresh
zero-quality link still does not assert streaming. Invalid packets and identity
replies cannot restore cleared items. Raw diagnostic records/counters are separate
from reset-sensitive model sensor history.

Linux Value/Gauge widgets therefore use MCU labels, units, current/min/max and
source lifetime, with advancing control identity and freshness checks. They do
not reconstruct extrema from raw radio diagnostics. See the
[sensor wire contract](../protocols/g11-autonomous-management.md#timers-sources-sensors-and-resets).

## Evidence and deployment limits

Native parser tests include 50,000 CRC-valid mutations. Combined actual-ARM tests
exercise decoder/engine/channel frames, independent expiry, fallback, Linux loss,
receipt counters, reset and clock wrap with simulated radio/peripherals. Numeric
tests compare 9,646 accepted configurations and 154,336 state transitions against
the pinned native setter; 434 configurations are refused. Reset comparison uses
51 checkpoints against native setters/function evaluation.

The revision-14 deployment used schema `0xeddcc26d6cb570ec`, MCU ELF
`36af66b3dbcf077438d611ebe2ca4acfdedd8295061ece65cbda34dc7103dae1`
and UI `31038c89e18e696471f66582de54c8a298abafcc46738f92d7ba5fd196781796`,
boot 122. Paired migration preserved configured and working-state bytes.
Its hidden holdover record shows 1,505 mixes in 3.010034 seconds, 499.994 Hz,
while the UI was suspended; observed maximum mix/management/lateness were
803/367/231 µs. These are finite RF-off observations, not worst-case bounds.

Sensor widget deployment at boot 120 and numeric deployment at boot 122 establish
installed software and ordinary autonomy. Value/reset/transform behavior uses
synthetic telemetry tests. Later [connected CRSF tests](../experiments/g11-crsf-electrical.md)
still recorded faults and no replies; no real Nano sensor or receiver-delivery
qualification follows from the deployed UI.
