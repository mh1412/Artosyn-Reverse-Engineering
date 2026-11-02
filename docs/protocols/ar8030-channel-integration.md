# AR8030 experimental channel and remote-synchronization integration

[Protocols](README.md) · [Channel-policy contracts](ar8030-channel-policy.md) ·
[Power control](../firmware/ar8030-power-control.md)

This page preserves the archived checkpoint sequence after hardware trial 161.
Its channel, synchronization and automatic-band work passed offline comparisons
and native synthetic tests, but repeatedly identified trial 162 as **not yet
run**. The last physical receiver state was restored stock camera video after
failed native level-11 acquisition. These experimental integrations are not
additional hardware acceptance of the later Open8030 release.

The later parity audit documents a smaller set of channel features integrated
into its receiver runtime. Those runtime limits and this earlier experimental
branch describe different source checkpoints. A helper's presence, an offline
RF adapter and a demonstrated physical channel change must remain separate.

## Checkpoint progression

Nineteen distinct versions of the power/channel narrative were compared, with
unchanged text accounted for by exact content comparison. They preserve the
following progression rather than nineteen independent experiments:

| Step | Added capability | Remaining boundary |
| --- | --- | --- |
| Peer-policy synchronization after 158 | Incoming subtype 8 and independent enable bits | Trials 159/160 had no incoming update or connected link |
| Login-sync dispatcher | Channel, PHY-filter and local-power flag dispatch | Initially only local-power callbacks were integrated |
| Channel-word and tuning helpers | Subchannel arithmetic, directional PLL mapping and copied configuration | No physical hop acceptance |
| Hop monitor and transaction | Confirmation, failure, full-word matching and rollback | Cross-band/auxiliary behavior remained bounded |
| Catalog and startup integration | Owned rows, attributed scores and initial selection | Internal rows did not implement the host catalog getter |
| Admission and pre-login requests | Independent advertised TX/current RX, state-1 requests | Assigned peer/session gates still required |
| Remote-policy integration | Channel/filter callbacks, retained effects across queue retries | Active auxiliary search remained open |
| Connected synchronization | Port-0 command 3 and periodic retry/ACK service | No over-air command/ACK or timing acceptance |
| Band-policy integration | Host request, capability gate and synchronized frame rebuild | Automatic-frame duration still used the three-command calculation |

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
5,772,500 and 5,777,500 kHz. Native parsing is atomic and accepts only powers
of two from 1 through 128 as the count; it does not reproduce original silent
fallback or byte narrowing on malformed settings.

RF owns a copy of the complete loader table and subchannel settings, including
when a fixed-channel RX context contains only one active row. The internal
resolver/tuner checks lifecycle and index bounds; deinitialized hosts cannot
use it. Stop/start, deinit/init and recovery preserve the saved settings. This
ownership does not imply that `GET_CHAN_INFO` succeeds through the host stack.

## Directional tuning and catalog arithmetic

The ordered tuning plan first identifies five-byte PLL shadow placement, then
band/path controls and RX hints. The caller calculates/validates the PLL and
writes those five bytes before the planned fields. Users 10/11 share shadow
`0x90a` in both directions. Normal users have nonuniform placement; TX0 is
`0x910`, RX0 `0x93a`, TX8 `0x900` and RX8 `0x905` relative to the baseband page.

RX users 0–10 receive `frequency/2000` and gain hints. User 10 or an empty gain
table selects 60; other users select the first threshold strictly above the
frequency, otherwise the final row. Input order is preserved even for unsorted
thresholds. User 11 has no hint writes. The early RF adapter retained fixed
broadcast/other gain values 32/60; full loader gain-table and TX band-power
integration were separate work.

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
An empty result uses channel 255. Native pointer/count/flag validation precedes
mutation, and original bit 26/per-row board-gain metadata was not integrated.

The archived comparisons covered 11,880 tuning cases with PLL/writes/power
helpers intercepted, 3,712 catalog cases and 1,494 bounded classification cases.
They do not execute whole boot or measure RF performance. The later adapter
populated scores from attributed measurements and used startup selection when
`init_chan=-1`; admission then replaced that preference with current RX and the
peer's independently advertised TX channel.

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

All intended tunes are validated before native state/MMIO effects. The compared
original callback could retain state changes despite a downstream tune error;
those are different failure guarantees. Auxiliary-search fields existed in the
planner but remained disabled in this RF integration.

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
nonzero subchannel. Timeout stages rollback to the last confirmed full word;
shared-queue contention retains rollback intent for a later poll. The native
request returns queue-busy directly rather than reproducing the vendor's
asynchronous failure message. Completion clears recent PHY error bitmaps and
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
input bytes. The experimental adapter retains accepted effects across identical
pending retries to avoid starving a shared queue. Its ACK covers the original
four bytes. Native also withholds completion if calibration rejects the level;
the original dispatcher ignores that setter return.

Login integration retains matching mode-0/slot-0 and AP-state-2 gates. The
supported channel/filter path owns catalog/cache state and the verified inactive
auxiliary fallback. Selector 0 can regroup and request a cached best channel;
selector 1 tries auxiliary search before normal mode/request effects. PHY10e
changes filtering, can select a replacement and refreshes auxiliary search.
Calling only the final tune/request would omit these policy effects.

Once connected, port-0 command 3 stages the same four-byte body for the rounded
DEV timer. The original path passes through MAC dispatch, LINK message `0x208`
and periodic service; it does not repeat connection or MCS initialization.
Native validates exact body length and session/control context before accepting
fragments. Queue contention retries on later ticks without another packet.
An explicit repeat after completion rearms application and ACK. A new request
replaces an unsent ACK, but cannot retract an already owned packet or accepted
hop. Recovery discards pending synchronization. These cooperative guards are
not proof of original asynchronous scheduler equivalence.

The retained connected-command comparison had 512 staging cases and 24 service
gates, including execution of the actual checksum function/table. Copy, mutex,
logging and posting helpers were intercepted; dispatch began after envelope
validation. Native tests additionally covered fragments, invalid controls,
foreign broadcasts, insufficient ACK budget and token invalidation.

## Automatic-band requests and frame-timing follow-up

Host command `0x02000012` takes one byte and returns an empty reply. The archived
adapter requires connected state 2 and admitted peer extension 5, returning
`-8` disconnected, `-15` without capability and `-12` for shared-FCH contention.
It queues command 8/subtype 9, TTL 16, preserving the complete input byte across
the first two payload bytes. Successful enqueue does not immediately apply policy
or establish remote delivery.

Incoming subtype 9 or outgoing expiry applies selector-0 policy and rebuilds
the DEV frame. Nonzero enables grouping and five commands; zero collapses groups
and installs three. Native validates the six-group bound, defers rebuilding
until RX DMA/startup-scan ownership permits it, and retains the latest pending
request. Timer/round producers wait during deferral. Both engines are held
before writes, with prior hold bits restored on success; hold failure stops RF
without publishing policy. Disconnect clears deferred work.

The original comparisons covered 544 host cases and 572 callback cases, including
all 256 request bytes in both directions. Native tests exercised TTL expiry,
DMA deferral, startup gating, invalid groups and failed holds. No hardware hop,
peer receipt or physical timing result followed those checks.

The band-policy checkpoint reported a 93,612-byte runtime, SHA256
`2268139c695931b0b441878f567f2fb9075e80063eb00b6a306ee87f9c8b9815`,
and 704,016 of 704,512 bytes allocated RAM, leaving 496 bytes in that build
accounting. This is not a measured runtime stack/free-memory margin. The
2,512-byte automatic-state owner reused padding for two pending-policy bytes.

At the band-policy checkpoint, the next task was to correct frame-duration
calculation: added
automatic slots existed, while `dev_frame_microseconds` still summed the
three-command frame. The later [installed-frame duration study](../firmware/ar8030-command-engine.md)
documents the correction and its offline validation; the
[automatic-scan study](ar8030-automatic-channel-scan.md) covers measurement,
timer and ownership integration across these checkpoints.
Wider channel/band commands, auxiliary/error-driven
hopping, normal acquisition at 11 and full normal-load/cold-start/recovery/video
acceptance remained open. This checkpoint cannot be labeled a working automatic
radio implementation from its successful build and helper tests.

For the wiki review, rebuilt checkpoint code reproduced 7,680 monitor-transition
cases and 14,076 valid frequency calculations; nine original invalid-index
cases were recorded separately. Original arming/monitor instruction bytes were
also checked to resolve the deadline field. The original emulator, automatic
runtime suite and hardware tests were not rerun during this review.
