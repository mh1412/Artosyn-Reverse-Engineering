# AR8030 acquisition, timing and resource management

[Protocols](README.md) · [Wire formats](ar8030-wireless-format.md)

This page records the matched 18.21.10 receiver reconstruction, primarily DEV
mode 0, assigned slot 0. It preserves distinctions between original-firmware
comparison, integrated replacement behavior and hardware validation. Later AP
and P1 results do not retroactively validate every helper described here.

## Slots and admission

Full AP control has eight peer records, numbered 0–7. Slot 8 in DEV reports
means unassigned. The structure suggests provision for multiple peers under one
coordinator, but does not establish eight concurrent receivers, equal airtime,
independent streams, mesh or multiple aircraft feeding one ground unit.

Candidate identities are alternative acquisition targets, not simultaneous links.
The implemented receiver path applies these gates:

1. Validate MAC/control checksums and extents.
2. Match current channel and relay policy. Outside pairing, the local identity
   must occupy position 0 in address extension 2, and the AP must match the
   configured peer or one of up to 100 host candidates.
3. Find compatible availability. Changed mode/bitmap yields a reconfiguration
   decision rather than an implicit hardware-profile change.
4. Require matching receive/transmit channel, admit the peer, suspend search and
   enter local state 1. A failed hardware transition leaves TX held.
5. Accept a login record matching local identity and token. The tested connected
   path requires slot 0, AP state 2 and `sync[0] == 0`.
6. Queue a command-4 synchronization ACK, enter local state 2 and enable receive
   MCS policy.

Local state 2 can precede actual ACK transmission. It is not proof that the AP
received that ACK, and unsupported synchronization flags are not acknowledged
as executed. External forwarding additionally requires no pending recovery.

Subsequent controls can invalidate the assignment through token, state, channel,
mode or bitmap changes. A full control identifying another peer does not
authorize forwarding its payload. Radio reacquisition also does not prove the
Linux video pipeline restarted correctly.

The [isolated-stock acquisition comparison](../experiments/ar8030-receive-acquisition.md#establishing-a-working-isolated-stock-reference)
reached state 2 after supplying the backend's volatile candidate-list command;
longer observer runs without it remained idle. Native receipt of a valid control
packet and offline READY decisions preceded implementation of this transition.
The [startup study](../soc/ar8030-receive-startup.md) separates search retries
from the later maintenance timer.

## Binding and persistence

The host handles button input, pairing commands, repeated result polling and
saved peer selection. The radio supplies pairing admission and the over-air
exchange. Binding therefore crosses both layers.

The 14-byte enable request contains enable, AP slot mask and three identities
that suppress pairing interpretation for those peers. DEV ignores the original
AP mask. Pairing uses a zero PHY identity and sets bit 6 of DEV-report byte 4;
it replaces normal peer/local-position filters but retains wire, channel, mode
and token validation.

The 164-byte result has a bitmap at byte 0, eight four-byte identities beginning
at byte 1 and eight 16-byte measurements beginning at byte 36. Only selected
entries are valid. The studied DEV publishes completion after matching login
reaches state 2, not upon merely hearing an advertisement.

The matched Ascent backend polls about every 20 ms within a 30-second budget,
checks repeated identities, disables pairing, selects the peer and saves its
Linux JSON. Another USB application capture used roughly 500 ms polling;
poll cadence is host policy, not an invariant of the radio protocol.

Physical tests progressed from an unsupported pairing command in stage 232 to
successful host-driven binding in stages 233/234. Saved selection survived a
receiver reboot and firmware reload. Cancelling unfinished pairing required
restoring the entire candidate-based acquisition policy; restoring only the
configured-peer filter could strand the prior link. The corrected cancellation
reacquired in about one second in the tested case.

This does not establish an encryption-key exchange. Drops, intermittent image
corruption and warm-load black-camera behavior remained separate issues in those
historical receiver stages.

## PHY TTL and shared command cache

For supported nonzero commands other than AP-time command 7:

- TTL 1 or 31 applies immediately.
- TTL 2–30 schedules after `floor((TTL−1) × frame_us / 1000) + 4` ms.
- TTL 0 does not schedule a new application.

TTL is frame-relative synchronization, not an IP hop count. There is one cached
message and shared timer, not a queue per received command. The compared sender
also has one pending slot; normal countdown transmits current TTL before
decrement, and a later tick at zero invokes its callback and clears the slot.
TTL 31 persists until cancelled.

Mode-3 comparison checks only the checksum low nibble; the high nibble carries
per-slot retransmission bits, and byte 2's high nibble supplies a disable mask.
Compared mode-3 admission/timer behavior was not equivalent to an integrated
mode-3 RF runtime.

## AP-time estimate

Command 7 reuses TTL positions for timestamp bits:

```text
AP_ms[27:0]  = LE32(payload bytes 3–6) & 0x0fffffff
AP_ms[31:28] = message byte 1 bits 7–4
```

Byte 2 bit 0 is excluded. Normal checksum and length checks still apply.
The DEV stores unsigned-32-bit `offset = AP_ms − local_ms`. Getter
`0x0100000c` returns LE32 `local_ms + offset` modulo 2³². Before synchronization
it returns `0xffffffff`, which is also a possible valid wrap-boundary value;
there is no independent validity bit.

Local milliseconds derive from a continuous 40 MHz counter divided by 40,000.
Stop/start preserves the offset; deinit/init clears it. Command 7 does not arm
or cancel the shared timer. Replacing a cached command prevents the old command
being reapplied on expiry, because command 7 has no timer callback.

The helper/getter matched 2,304 original-instruction emulator cases with clock
and hardware adapters. That establishes the compared arithmetic/control behavior,
not a measured hardware synchronization error or video latency.

Command 6 separately updates one-byte PHY identities: payload byte 3 low nibble
selects user 0–9, bit 4 selects TX when set, and byte 4 supplies the value. Those
identities are not the four-byte admission identity. DEV idle skips RX changes.

## MCS adaptation

Receive policy uses an eight-sample SNR/error window and configured rows.
Internal MCS indices equal configuration values plus two; they are not a
universal IEEE 802.11 numbering system.

An outgoing command-3 request uses TTL 31, length 2, assigned slot and
`internal_mcs | 0xc0`. The high bits are a bandwidth sentinel, not evidence of
general bandwidth negotiation. A valid FCH mapping to the requested MCS through
board encoding keys acknowledges the change. An unsolicited configured MCS can
also update the active policy row.

Advertisement phases alternate in groups of five pre-encode calls. A wait
timeout can request RX reset without claiming the MCS transition succeeded.
Cancellation clears command state rather than inventing acknowledgement.

Incoming command 3 addresses DEV transmit MCS. The studied integration accepts
slot 0 and internal indices below 16 with valid normal encoding keys. Increases
apply immediately; a decrease waits for six user-0 interleaver events and applies
on the seventh. Other users do not advance that drain, and another target is
ignored during a pending decrease.

Attempt-history bits can advance even when packet construction later fails;
they are not completed packets or peer acknowledgements. Retained-buffer retry
ownership, special keys and sustained adaptation under interference remained
unverified by that copy-mode implementation.

## Hardware preparation and measurement

The normal DEV frame has broadcast RX, a special-user interval and local TX,
each with preparation. Metadata records raw user, mode, bandwidth, frequency,
calibration bank and buffer flags. A preparation cursor advances and uses the
original fallback matching rules; unmatched metadata leaves cursor −1 with no
preparation writes. TX completion alone cannot establish correct preparation.

Changing calibration bank loads five bytes. An initial cached value of 3 also
enables compensation and initializes two fields. Frequency-subrange changes
drive separate board controls; at the observed 5,770 MHz boundary the callback
programs six RF controls. These caches and TX-buffer ownership are distinct.

Loaded compensation survives normal stop/start, reinitialization and native
recovery. Probe/default/calibration/diagnostic transitions invalidate assumptions.
Only bank 2 had native calibration data in this receiver stage; unavailable banks
were not silently replaced with bank 2. Offline synthetic-bank comparisons are
not hardware validation of other bands.

The special interval uses raw user 13 / logical RX user 11. It can sweep its
own measurement frequency while active link RX/TX stay fixed. Tuning programs
five PLL bytes, band/path fields and retained metadata; this logical user skips
ordinary RX gain hints.

The studied replacement swept 42 configured channels while the stock host chose
four enabled work channels. Accepted measurements updated ten raw samples and
ten rolling means per channel plus integer scores. Ambiguous tune/prepare events
were excluded, so the results did not establish equivalent cadence or unbiased
RF measurement. Ranked four-channel backup policy and a 3,000 ms timer were
compared at helper boundaries, without proving a physical hop or peer receipt.

The [channel-policy study](ar8030-channel-policy.md) details the host catalog
dependency, score arithmetic, staged changes and monitor completion rules.

## Search, acknowledgements and recovery

For frame duration `F` microseconds, search dwell starts at
`floor(F/1000)+2` ms and alternates at list wraps with
`2×(floor(F/1000)+1)` ms. A first valid candidate header gets a two-frame
DMA/admission grace period; unrelated headers cannot extend it indefinitely.
Pending DMA or admitted peers suspend retuning.

Frame duration sums six intervals, rounding each 640 ns hardware count
separately to microseconds. It is not the video frame period.

| Signal | What it establishes |
| --- | --- |
| Login command-4 ACK | Supported synchronization bytes acknowledged by DEV |
| Requested MCS seen in FCH | Receive policy observes the requested coding mode |
| Local TX retirement | Hardware buffer may be reused |
| Application video decoded | Delivered stream was usable for that sample |

The retransmission watchdog observes four-bit TLV request IDs, not the MAC
sequence. A zero threshold disables it. Repeated IDs outside `(last+1) mod 16`
or `(last+2) mod 16` can disable TX temporarily before a two-call re-enable phase.
This is not a complete software ARQ implementation.

FCH loss tracks valid observations for logical RX user 8. Byte truncation/capping
in threshold calculation and delayed cooperative polling mean it is not a simple
exact one-second timeout. A delayed poll counts once.

Loss or invalidation stops new submissions, then deferred recovery stops radio,
rebuilds saved configuration while preserving host candidates, and restarts RX.
Receive reset waits for pending DMA. Physical peer silence/restart and all
pending-DMA failures require broader evidence than candidate removal/readmission.

## Internal-service ownership findings

These findings came from original-instruction comparisons, not a declaration
that every service was integrated into the replacement runtime:

- The internal writer suppresses nonempty writes above 32 queued bytes, or above
  262 when its mode field is 1; equality permits writing. Suppression returns
  zero and the route helper releases the packet without retry.
- Queue discard removes available bytes but retains already acquired occupancy.
  It occurs before the capacity check, so a rejected write can still change the
  queue. Nonblocking writes can be partial; an error after earlier passes need
  not mean no bytes were published.
- Worker age above 50 ticks increments a diagnostic counter rather than expiring
  a packet. The clock adapter determines the tick unit; it is not proven 50 ms.
- Synthetic copying routes retained references after compared worker drains.
  The 49 recovered built-in routes did not request those actions; this is not
  evidence of a general stock memory leak.
- Interface construction can publish partial resources before later failure.
  Some task/mutex errors were ignored at the compared boundary. A non-UART
  handle-allocation failure after socket open could return zero without closing
  that stream at that boundary. Success return alone is insufficient evidence
  that all services exist.
- Event delivery copies declared payloads into transport-owned storage.
  Unsubscribe prevents new publication but preserves queued events. The studied
  cooperative transport limits were 16 events and 4,096 payload bytes.

Native bounds guards intentionally differ from unchecked original behavior.
Emulator adapters at RTOS/hardware boundaries do not prove scheduler timing,
interrupt behavior or production lifetime correctness beyond the compared scope.
