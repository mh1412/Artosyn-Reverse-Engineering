# AR8030 channel and remote-synchronization reconstruction

[Protocols](README.md) · [Channel-policy contracts](ar8030-channel-policy.md) ·
[Power control](../firmware/ar8030-power-control.md)

This study records stock Ascent 18.21.10 channel and synchronization behavior
recovered with original-instruction comparisons. Synthetic state and intercepted
hardware callbacks bound the findings; they do not demonstrate physical hops.

## Channel words and subchannels

A 16-bit channel word uses its low byte as catalog index and high byte as
subchannel ID. The selected row's base frequency is masked to 24 bits. Disabled
subchannels, ID zero or ID above the configured count use that base unchanged.
Otherwise, with frequency and bandwidth in kHz:

```text
frequency = base + offset - main_bandwidth/2
            + (2*subchannel_id - 1)*main_bandwidth/(2*count)
```

Division is integer division and the final arithmetic retains unsigned-32-bit
wrap. The separate subchannel-bandwidth setting does not enter this formula.
The caller must reject unsupported resolved frequencies before writing RF
registers. An arithmetic result is not evidence of physical tuning support.

Absent settings default to disabled, 10 MHz main width, 1.25 MHz sub-width,
count 4 and offset 0. The matched JSON instead enables 20 MHz/2.5 MHz/count 4.
Its channel 24 base of 5,770,000 kHz resolves IDs 1–4 to 5,762,500, 5,767,500,
5,772,500 and 5,777,500 kHz. Original fallback and byte narrowing on malformed settings need separate
interpretation from successful examples.

## Directional tuning and catalog arithmetic

The ordered tuning plan first identifies five-byte PLL shadow placement, then
band/path controls and RX hints. The caller calculates/validates the PLL and
writes those five bytes before the planned fields. Users 10/11 share shadow
`0x90a` in both directions. Normal users have nonuniform placement; TX0 is
`0x910`, RX0 `0x93a`, TX8 `0x900` and RX8 `0x905` relative to the baseband page.

RX users 0–10 receive `frequency/2000` and gain hints. User 10 or an empty gain
table selects 60; other users select the first threshold strictly above the
frequency, otherwise the final row. Input order is preserved even for unsorted
thresholds. User 11 has no hint writes.

The catalog classifies frequencies at boundaries 1,000,000 and 4,300,000 kHz.
Its preferred flag uses strict endpoints for 2,400,000–2,483,500,
5,150,000–5,250,000, 5,250,000–5,350,000, 5,470,000–5,725,000 and
5,725,000–5,850,000 kHz. These are original software classifications, not
regulatory permission or measured board capability. Initialization classifies
the full loader word before storing its low 24-bit frequency; later regrouping
reads that stored frequency.

Group thresholds are consumed in order, with equality advancing the group.
Eligibility combines band mask and, optionally, the preferred flag. Selector 3
is a wildcard for group or band. Group minimum compares signed 32-bit scores;
band ranking compares signed low-16-bit scores and retains input order on ties.
An empty result uses channel 255.

The archived comparisons covered 11,880 tuning cases with PLL/writes/power
helpers intercepted, 3,712 catalog cases and 1,494 bounded classification cases.
They do not execute whole boot or measure RF performance.

## Hop request, callback and confirmation

The internal assigned-DEV request API also accepts pending login in state 1.
Unassigned peers are rejected. Zero means accepted or already selected, not
confirmed. After validating the complete channel word and calibrated frequency
range, it queues command 4, TTL 16, with three bytes:

```text
0x80 | assigned_slot, channel_low, channel_high
```

Queue acceptance enables the configured D30 synchronization field. The header
waits for the next interleaver. Outgoing expiry applies the channel callback
before writing the cleared header. In the supported DEV path it updates slot
and broadcast RX, tunes logical user 8 RX and emits event 2. Incoming slot-0
bit 6 tunes assigned-slot TX; bits 4/5 independently change broadcast TX/RX.
Event 2 is `{0, direction, new_base_index, previous_base_index}`. Subsequent
control checks use the updated broadcast indices.

The original callback can retain state changes after a downstream tuning error;
its effects are not transactional.

### Correct hop deadline field

The deadline is **`uint8(missed_count + 16)`**, not loss-threshold plus 16.
The matched DEV node contains the monitor object at offset 152, and logical
user 8's five-byte entry starts at `152 + 8 + 8*5 = 200`:

| Node offset | Meaning |
| --- | --- |
| 200 | Enable/lock/force/pending flags |
| 201 | Loss threshold |
| 202 | Current missed-observation count |
| 203 | Low channel byte |
| 204 | Hop deadline |

Original `0x23070e` reads node+202, `0x230716` adds 16 and `0x230718` stores
node+204. Separately, monitor initialization stores threshold at entry+1 and
zeroes entry+2; periodic processing increments entry+2. These bytes resolve
the inconsistent “threshold” label in the later helper/audit. A comparison
using a misnamed input can preserve byte operations without establishing that
input's semantic meaning.

Arming does not clear the prior hop-success streak. Three positive observations
confirm a pending hop; a negative observation while locked resets that streak.
Deadline equality can report failure before the ordinary loss callback. Normal
or forced loss, and completion of initial two-observation lock acquisition, can
clear pending state without producing a hop result.

Original confirmation retains only the low channel byte. The transaction
compares it against the **full pending word**, preserving the mismatch for a
nonzero subchannel. The vendor reports request failure asynchronously; queue acceptance is distinct
from completion. Completion clears recent PHY error bitmaps and
consecutive/max counters while preserving sample rings, rollups, MCS history
and scan scores.

## Login synchronization and connected updates

Original `0x232928` dispatches a four-byte synchronization value. The archived
8,192-case comparison covered 4,396 complete outcomes and 3,796 pending outcomes
with channel, PHY queue, setter and ACK boundaries intercepted.

| Flag bits in byte 0 | Effect |
| --- | --- |
| 0/1 | Channel selector 0; optional value from byte 1 bits 1–2, no automatic-mode argument |
| 2/3 | Channel selector 1; optional automatic bit from byte 1 bit 3 and value from byte 2; 255 means no value argument |
| 4 | PHY message `0x10e`, byte 1 bit 4: channel filtering/selection, not local power mode |
| 5 | Local automatic/manual mode from byte 1 bit 5, before level selection |
| 6 | Apply byte 3's level to physical user 0 only if resulting mode is manual |
| 7 | Unresolved; prevents complete acknowledgement |

Original mode switching only toggles stored values 0/1; other nonzero host
policy bytes survive either requested mode. Callbacks may apply some effects
while others remain pending. Original retries repeat operations from unchanged
input bytes. The original dispatcher ignores the power setter's return value.

Once connected, port-0 command 3 stages the same four-byte body for the rounded
DEV timer. The original path passes through MAC dispatch, LINK message `0x208`
and periodic service; it does not repeat connection or MCS initialization.

The retained connected-command comparison had 512 staging cases and 24 service
gates, including execution of the actual checksum function/table. Copy, mutex,
logging and posting helpers were intercepted; dispatch began after envelope
validation.

For the wiki review, rebuilt checkpoint code reproduced 7,680 monitor-transition
cases and 14,076 valid frequency calculations; nine original invalid-index
cases were recorded separately. Original arming/monitor instruction bytes were
also checked to resolve the deadline field. The original emulator and hardware tests were not rerun during this review.
