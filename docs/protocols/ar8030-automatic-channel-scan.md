# AR8030 automatic channel scanning and service lifecycle

[Protocol index](README.md) · [Channel selection](ar8030-channel-policy.md) ·
[Command engine and frame duration](../firmware/ar8030-command-engine.md)

This article reconciles the archived stage-162 scan checkpoints against the
Gnd485 18.21.10 original runtime. It covers progression from standalone helpers
to an experimental continuous DEV adapter. Later additions supersede earlier
“not integrated” statements only within that branch. No hardware trial 162 ran;
the software comparisons do not establish physical sample attribution,
automatic hopping or full replacement-firmware acceptance.

## Three different measurement histories

| History | Input and use | Reset/readiness boundary |
| --- | --- | --- |
| Independent base-channel survey | Logical user 11 / raw 13 power measurements; startup selection | Shared ring advances on the last eligible row; ready does not guarantee fresh samples for every row |
| Local and peer PHY history | Local user 8 and connected peer report; quality/error policy | Age, sample index and error flags have separate reset rules |
| Automatic search reports | Logical user 10 / raw 12 quality/gain samples; ranking and round decisions | Live reports, report cache, scratch samples and ranking counters have distinct ownership |

The independent survey keeps latest measurement, ten raw samples and ten means
per base channel. It replaces the current raw entry, averages using a 64-bit
unsigned sum, stores that mean and averages the ten means. Band/group results
are evaluated after ten rounds. Dropped earlier rows can leave older entries
even when the shared round counter says ready. Startup chooses the configured
band or a biased minimum-average result. Score normalization is a recovered
integer function, not a calibrated dB measurement; band sums can wrap before
signed division. AP/subchannel measurement ingestion is outside this base survey.

PHY history instead has eight 16-byte samples and eight rollups, advancing its
index before insertion. Error sums narrow to 16 bits; gain uses the average of
the smaller of the two gains per sample. Unsigned millisecond age greater than
200 clears index/readiness while retaining data and error bitmap. Severity is
the greater of recent-eight error popcount and bounded consecutive-errors minus
one; the consecutive count saturates at 255.

An in-flight hop completion clears local/peer error bitmap and consecutive/max
error state even for failure or a full-word confirmation mismatch. Samples,
rollups, MCS and scan history survive. The native adapter owns local user 8 and
peer slot 0 rather than all original per-user history pointers. Session and
exact-length checks precede peer-report mutation.

## Startup survey and receive-search handoff

Startup configures logical user 11 RX on the first eligible base channel. The
observed setup includes a 2,500-count wait, IRQ 5 enable and optional raw-13
power/raw-14 wait descriptors. DEV bypasses the AP extra-catalog survey even
when subchannels are enabled; its base-channel survey still runs when required.
Manual initial-channel configuration skips that survey.

While unresolved, startup holds payload admission and automatic search. Once a
selection exists, the transition holds both engines, resets/gates the old state,
installs the normal frame and timing, prepares RX8 search, clears stale IRQ state
and restarts reception with TX held. **The survey winner does not replace the
existing receive-search channel at LINK `0x200`.** Best selection and current
RX search remain separate. Later admission can make current RX the preferred
channel while the survey result remains in history.

Warm stop/recovery invalidates sample tags but retains survey history and
selection. Explicit initialization resets and surveys again. A startup timeout
stops/holds RF and rejects another start until fresh configuration. Failure of
the initial idle wait must not write a new frame.

Original IRQ 5 reads and byte-swaps the power word and posts a four-byte PHY
message without a channel tag; its worker reads channel state separately. The
native adapter therefore arms one tag only after a successful tune and raw-13
RX preparation. Pending IRQs around tuning/arming are discarded. Shared-bank
user10/user11 writes, contradictory body observations and context changes
invalidate the tag. A tag is consumed once. These guards cannot prove that
real coalesced IRQs or repeated hardware contexts are attributed correctly.

The shared scheduler advances/wraps its logical cursor in observed modes 0/3,
with a raw-user fast path and first matching user/mode fallback. Inhibition does
not suppress cursor, board or gain effects. An eligible raw-13 next command
advances the independent channel; an empty catalog preserves its channel word.
TX board-calibration preparation can occur with TX held. Original helper
comparisons and synthetic held-write checks do not establish physical cadence.

## Admission, selection and a corrected indexed read

The mode-0 admission adapter checks the advertised TX/current-RX context and
lock token before held TX tuning. It records slot/token before remote sync to
avoid a dependency cycle; assigned states 1/2 can request hops while an
unassigned node cannot. Failed handshakes retain TX hold. A connected broadcast
context change enters recovery rather than silently retuning the session.

Selector 0's requested value 3 is a special recovered memory access: it reads
the halfword at `0x26c564`, the **low half of band 0's score**. It is neither a
fourth band candidate nor group 0's candidate; the latter is at `0x26c572`.
Earlier checkpoint prose misidentified this halfword. Group selector 3's
wildcard meaning in other functions does not change this indexed read.

Search traverses eligible original catalog indices, preserving sparse indexing
and wrapping at the end. Native configuration rejects an empty eligible set or
an excluded fixed channel. Repeated accepted synchronization can retain policy
effects; its ACK is not a second hop-confirmation event. Report byte 5 uses the
low byte of the preferred/current RX field. See
[hop and remote-sync integration](ar8030-channel-integration.md) for the full
request, confirmation, retry and rollback boundaries.

## Automatic report cache and ranking

Automatic state covers 128 channels and six groups. PHY `0x111` accepts a mutable
14-byte report and normalizes byte 0 to the base-channel index. Nonzero byte 1
copies the full report and increments cached sample count only while below ten;
an already larger count is retained. Zero clears the cached report/count. Both
paths mark ranking dirty. Bytes 2/3 and 12/13 remain opaque copied data.

Ranking rebuilds only when enabled and dirty. It clears dirty, initializes best
channels to 255 and best group to 6, and resets group quality/attempt/auxiliary
fields. Eligible valid rows require at least two samples. Quality is the maximum
of four unsigned 16-bit values. A candidate wins with a sample-count advantage
of at least two regardless of quality, or with nondecreasing count and strictly
better quality. Equal comparisons retain the earlier row.

Selection starts with the requested group's cached candidate. A different valid
best group can replace it when the candidate is 255, equals the current channel,
has fewer attempts or fails the final three-halves quality comparison.

The original chooser has an unusual indexed-memory behavior: indexing its
quality base by **channel**, rather than group, reads group quality for indices
0–5, group attempts for 6–11, group auxiliary values for 12–17 and channel
attempts at index minus 18 thereafter. The native implementation models those
reads explicitly without out-of-bounds C access. Replacing them with a presumed
group-quality lookup would change the compared behavior.

## Round counters and pending requests

Round counters count qualifying observations, not transmitted hop requests.
For an invalid current channel, valid eligible rows advance only when all four
qualities exceed 1,000. Reaching 300 requests a channel and clears counters;
the traversal can then continue and increment later rows again.

For a valid current channel, a candidate's maximum quality is compared with
`(current_max * 3 / 2) & 65535`. Exceeding that truncated threshold increments
its counter; reaching 300 requests immediately, clears channel counters and
returns. A value below current maximum clears its counter only if it did not
already pass the first, potentially wrapped comparison. Intermediate/equal
values retain it. Increments wrap at 16 bits before threshold testing.

Other-group counters advance when their temporary best quality exceeds the
current group's; current-group counters are retained. At the threshold, selection
uses the greatest channel counter in the selected group's eligible traversal,
with current channel as incumbent. It clears channel/group counters even when
the winner does not change. Group selector 3 remains a wildcard here.

The fallback uses the latest local PHY sample, not its MCS average, plus keep
thresholds and ranking state. Original request-helper rejection still permits
counter resets: request generation is not queue acceptance. The RF adapter
retains the latest chosen channel on contention and retries it without rerunning
the decision/reset sequence. Its additional cooperative idle retries do not
prove original RTOS scheduling equivalence.

## Automatic scan cadence and sample ownership

Logical user 10/raw 12 is distinct from the independent user11/raw13 survey.
Preparation runs after the shared scheduler. Observed modes 0/3,
inhibition, absent current command or a next command other than raw 12 leave
scan state unchanged. Phase 0 is idle; phases 1–3 advance toward 4. At phase 4,
the adapter selects the first eligible row, copies directional bandwidth, tunes
and starts repeat counting. Subsequent preparation repeats until dwell expires.

On RX completion it aggregates/posts PHY `0x111`, then advances. Exhaustion
clears phase, holds the selected direction and posts RX round event `0x110`.
TX performs neither RX aggregation nor RX round posting. Failed first-channel
lookup retains the preceding channel; channel iteration uses byte wrap.

PHY `0x101` capture accepts user 10 only in phase 4 or above. It copies ten bytes
to `scratch[count % 20]`, then increments an eight-bit count, including wrap at
255. Aggregation divides sums of four 16-bit qualities and two byte gains by
that count, writing report bytes 4–13 while preserving bytes 0–3. Original
zero-count unsigned division produces all ones; the later cache invalidates the
zero-count report.

There is a real model boundary: 20 scratch slots cannot safely supply an
arbitrary 255-row aggregation. Large original-oracle fixtures seeded adjacent
synthetic memory. Native checks capacity and rejects count 21 before mutation;
it does not reinterpret those fixtures as proof of a 255-slot hardware buffer.

Original IRQ 2 samples when `(raw_user & 13) == 12`, allowing raw users 12 and
14. It preserves/restores the relevant selectors, reads four quality halfwords,
and uses three independent E60 reads plus one E64 read for two wrapped gain
sums. The original posted message includes two scratch-residue bytes beyond the
ten meaningful bytes; native does not reproduce that padding. Raw 14 remains
diagnostic-only in the native attribution path.

The adapter requires a successful user10 tune, raw12 RX preparation and stable
raw12 RX body context without an intervening IRQ7. Shared user10/user11 tuning,
frame replacement and holds invalidate attribution; stale IRQ2 is acknowledged
around tuning. Multiple valid samples may be accepted, but overflow aborts.
Candidate state stays private until frequency/context validation; failed
next-tune validation rolls back aggregation. Completed reports are cached before
the next tune, and RX is held before its round callback.

Directional preparation copies only user10's three-bit bandwidth from the
role-specific source: DEV TX0/RX8, opposite role TX8/RX0. It does not copy a
complete RF profile. Metadata updates use the first matching raw12 direction;
absence of metadata does not suppress the underlying field write.

## Scan negotiation, timer and lifecycle

Band service `0x21eb9a` requires a nonempty connected-slot bitmap and policy
flags bit 1. Otherwise it clears phase/counter and returns 255. A nonforced call
outside phase 1 returns 254. Eligible calls enqueue command 8, TTL 8 and a
four-byte subtype-10 body; the queue-attempt path returns zero even when busy.

| Queue result | Band phase/counter effect |
| --- | --- |
| Accepted | Phase 2, counter zero |
| Forced attempt busy | Phase 1, counter zero |
| Nonforced attempt busy | Existing phase/counter retained |

The payload stores subtype 10 in the low nibble and scan count's low seven bits
at bits 5–11. Original stack residue supplies bit 15 and bytes 2/3; native uses
an explicitly initialized seed. Incoming start selects RX; outgoing expiry
selects TX. The callback finds the first physical raw12/opcode5 slot, changes
its mode and, when present, its predecessor's preparation. It updates the first
matching logical entry, sets scan dwell/phase/direction, and releases user10
before publishing band phase RX3/TX2 and counter zero. No matching physical slot
still starts state. RX clears only live-report byte 1 for all 128 channels;
neither direction clears every cache or scratch buffer.

Timer callback `0x21ec76` forces service immediately in phase zero. Otherwise it
increments a 16-bit counter and forces service only when the result is strictly
greater than its threshold. Threshold 80 therefore fires on call 81 from zero;
threshold 65535 never expires through this comparison. Connectivity/policy
checks occur inside service, so a disabled nonzero phase is not reset early.

The recovered original timer uses period 50, repeat enabled and threshold 80.
At the observed 40 MHz clock, its 40,000-count core tick implies nominal 1 ms
ticks, 50 ms callbacks and 4,050 ms for that 81-call nonzero-phase interval.
Repeats advance from the preceding deadline. Scheduler suspension and callback
latency prevent treating these configured intervals as measured RF timing.

Native service uses 2,000,000-count deadlines, processes at most eight overdue
callbacks per poll and retains remaining debt. Warm restart establishes a fresh
deadline without charging stopped time while retaining applicable counters.
User8 EOP runs pending-hop work before nonforced scan service; alias/duplicate
observations are gated. The bounded poll policy is an implementation choice,
not demonstrated RTOS fairness equivalence.

Disconnect handling holds/drains RF before changing ownership. Original PHY
`0x305` state 2 sets a slot bit. State 0 clears that bit, band phase/counter and
peer-information flag, resets ranking, applies disabled-auto policy and rebuilds
the frame even without a preceding bit or a last-peer transition. Ranking reset
preserves dirty, group quality/attempt/auxiliary state, live reports and scratch;
its two observed `808[15]=1` writes both target TX. Failed native DMA draining
preserves ownership rather than publishing a completed disconnect.

Peer extension 5 indicates presence: the original reader sets its flag even when
the wire value is zero. Accepted mode-0 admission transfers that capability to
RF state; rejected/failed admission preserves prior state, and successful
disconnect clears it. Capability is separate from enabling automatic policy.

Loader configuration retains up to six ordered frequency thresholds. They are
not sorted automatically. Six thresholds can produce unsupported group 6, so
native validates all resulting groups before mutation. A sixth threshold above
the highest channel can remain valid. The original loader saves threshold count
while setting **active count to zero**: cold classification is group 0 even
with auto configured. Explicit selector-0 enable activates grouping; login mode
`-1` does not. Admitted stop/disconnect disables auto; warm restart retains that
state, while deinit/init restores loader policy with cold grouping.

Recovered defaults include SNR 3880, scan count 10, scan/round INR values 3,
gain threshold 0, up tuple `{1286,5,1}` and down tuple `{1024,5,1}`. The two
keep thresholds are zero when their object is absent and default to five when
it is present. SNR fields are 16 bits; most scalar fields are bytes and frequency
thresholds are 32 bits. The native loader rejects malformed, negative or
out-of-range values atomically. Raw interval fields retain their observed values;
these defaults alone do not establish their physical units.

## Validation scope

The records retain separate original-instruction comparisons, including 7,808
shared-scheduler cases, 777 PHY-history/severity/reset cases, 848 ranking cases,
512 report and 545 round cases, 2,560 IRQ2 sample cases and 2,100 band-timer cases.
Helpers and hardware accesses are intercepted at each oracle's stated boundary.
Synthetic RF tests add held writes, rollback, stale-sample rejection, queue
contention, timer wrap and overdue-callback handling. Case counts from separate
checkpoints are not independent hardware runs.

The later [physical-duration follow-up](../firmware/ar8030-command-engine.md)
corrects the earlier three-command timing calculation after automatic slots
were added. Its final build uses all 704,512 bytes of allocated RAM. These
advances leave normal level-11 acquisition, physical IRQ attribution, automatic
RF timing, auxiliary/multichannel/error-driven hopping and broader normal-load,
cold-start, recovery and video acceptance unresolved at the archived checkpoint.
