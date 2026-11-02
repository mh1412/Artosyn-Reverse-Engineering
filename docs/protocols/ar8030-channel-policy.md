# AR8030 channel measurement and selection

[Protocols](README.md) · [Link management](ar8030-link-management.md)

This study covers recovered 18.21.10 channel-control contracts and the historical
Open8030 DEV reconstruction. Offline-tested policy helpers and their callbacks
must be distinguished from the smaller set integrated into that receiver's
runtime. The later AP release does not retroactively validate each helper.

## Host catalog dependency

The stock Linux backend first requests channel information with `0x0100000a`.
Only a successful response populates its cached frequency catalog. It then
matches desired frequencies to catalog indices and sends a nonempty work list
with `0x02000023`. An empty match suppresses that setter entirely.

In the compared session, stock had 42 catalog frequencies and four enabled
indices reported by getter `0x01000012`:

| Index | Frequency |
| --- | --- |
| 15 | 5,839 MHz |
| 23 | 5,740 MHz |
| 24 | 5,770 MHz |
| 25 | 5,805 MHz |

Three queries agreed. These are session selections, not immutable board defaults.
The native backend desired the same frequencies, but the unsupported catalog
getter left its cache empty. Its audit consequently contained no work-list
setter even without overflowing. An earlier missing operation prevented the
later request; observed native traffic alone cannot inventory required behavior.

## Catalog and work-list semantics

The recovered catalog/work-list helpers were compared in 1,645 original-instruction
cases. At that stage they remained outside the runtime build and dispatcher.
Policy, AP tuning and auxiliary-reset callbacks were intercepted, and processed
scores were supplied inputs.

Consequential work-list rules include:

- Invalid or forbidden indices do not enable entries, but the first supplied
  index can still become a fallback/AP tune request. The callback must validate it.
- Membership checks use the full 16-bit current channel. Later group lookup uses
  its low byte. Automatic policy takes precedence over membership.
- Group selector 3 matches any enabled channel. A group's winner is cleared only
  if no enabled entries remain; otherwise the existing winner survives.
- AP locking surrounds the conditional broadcast-channel change. Auxiliary-scan
  reset follows group updates even when no channel change was requested.

Original unused reply bytes retain prior buffer contents; the independent
serializer deliberately zeroes them. Additional native structural validation
rejects unsafe inputs before mutation. Those are documented differences.

## Independent measurement path

The normal DEV command frame comprises broadcast RX, independent measurement and
local TX. The builder writes preparation/body entries, user nibbles, terminator
and displaced pre-command fields while execution is held. It retains up to
20 metadata entries. Retuning updates frequency and calibration bank; profile
updates change the first matching bandwidth/RX-flag entry.

Preparation selects metadata using the fast raw-user check and fallback
raw-user/hardware-mode match. Unmatched notifications perform no preparation
writes. TX entries select calibration, while separate board-region and
calibration caches suppress repeated writes. Body notifications leave the cursor
unchanged and apply the next TX entry's bandwidth look-ahead. The runtime stage
used only its fixed mode-0 frame and available calibration bank 2; other banks
were not substituted with bank 2.

The body notification before raw user 13 advances measurement logical RX user 11.
Its five PLL bytes occupy the special-user shadow at `0x90a`; link RX8 and TX0
retain their own shadows. Metadata follows the measurement tune. Acquisition on
a fixed link channel does not discard the configured measurement catalog.

The polling implementation attributes power samples only after its
tune/prepare/sample checks. Pending, duplicate and unconfirmed events are
excluded. One early window recorded 38 attributed samples, 442 exclusions and
158 command-consistency failures. Stock and native link PLLs stayed at 5,770 MHz,
but stock measured four selected frequencies while native swept its configured
42. Correctly changing PLL shadows did not establish equivalent sampling policy
or timing.

## Score processing

Each channel has ten raw samples and ten rolling means. An admitted sample
replaces the raw entry at the shared ring index. The mean of all ten raw samples
replaces that index's rolling mean; the mean of all ten means feeds the recovered
integer score conversion. Both sums use unsigned 64-bit arithmetic.

The shared index advances when the sampled channel has no later enabled entry.
Ten such rounds establish readiness. Ranking uses signed scores with stable
ties. Band-score sums retain 32-bit wrap; startup band selection has a signed
bias, and group 3 remains a wildcard.

The compared helper emits RX-hold and selection-message requests without
executing them. The historical runtime collected scores but did not enable
automatic startup selection or group policy. Context rebuilds cleared its
histories/readiness; complete stock lifecycle retention was not established.

Diagnostic `0x015a0027` version 2 is 392 bytes: the previous 53-word prefix,
ring index, readiness, completed-round count and 42 signed scores. A zero score
with zero sample count denotes an unmeasured channel. Sample counts are not
scores, and applying stock arithmetic to filtered events does not establish
unbiased noise measurement, calibrated units or equivalent sampling cadence.

## Backup selection and staged requests

Backup ranking selects up to four enabled channels in a group using the signed
low 16 bits of each score; ties keep catalog order. If nothing qualifies, it
returns the current channel even if disabled or outside the requested group.

A channel request updates pending state before posting destination 1/message
`0x108`. Selector 1 records a 16-bit pending channel; other selectors retain its
low byte as broadcast channel. Successful posting sets an additional flag;
failure leaves the pending state. Posting is not an immediate retune.

Auxiliary reset clears six group counters and 128 channel counters, writes
RF field `0x08080f01 = 1` twice, then resets six winners, a selected-group marker,
a state flag, 128 opaque 14-byte records and 128 marks. These records are separate
from the noise-score histories; their complete meaning remains unresolved.

## Selector-1 policy

Manual selection compares the full 32-bit request to the current 16-bit channel
before narrowing the posted request. A queue-post failure is not returned as
policy failure: the original reports success while preserving pending state.

With auxiliary hopping enabled, requesting mode 2 refreshes four backups. Each
chosen channel has bit 8 set; unused entries are `0x00ff`. DEV sends internal
command 10 to slot 0; AP sends to enabled slots. A 3,000 ms timer follows the
sends, even if sending fails or AP has no enabled slots. Entering hopping can
instead request the current TX channel through PHY control, depending on flags.

Leaving hopping requests the explicit low channel byte or the auxiliary channel.
Successful enqueue updates local channel/band/group from the auxiliary channel,
even if the explicit request differed. Ordinary errors leave those fields
unchanged; special result `-2` falls through to manual policy. These are enqueue
decisions, not peer acknowledgements.

## Transaction and completion

Worker messages `0x108` and `0x109` are internal request/completion messages,
distinct from host IDs and over-air internal commands. The request locks role
state and queues PHY command 4 with TTL 16 and a three-byte payload. DEV encodes
slot and requested channel; AP additionally considers selector, mode, flags and
relay configuration. Already-selected DEV requests and selector-0 AP broadcast
requests can return without sending.

An enqueue error posts failed completion after unlocking. Selector-1 completion
clears inflight state, its two-byte history marker and two counter lists when
inflight was set. Success commits only if the complete 16-bit returned channel
matches the pending value; a stale success can reset bookkeeping without changing
channels. Failure with error zero stages rollback to the current channel. Other
errors do not. A failed rollback post still leaves pending request state.

Other selectors use broadcast flags and preserve the separate history marker.
The returned full word must match the stored broadcast byte; equal low bytes
alone do not suffice. Callback arguments previously labeled user/port were
identified as command/TTL: `8, 8` denotes PHY command 8 with TTL 8.

## Channel-change callback and monitor

The compared PHY command-4 callback updates state before tune/notification calls.
Failures from those calls do not undo the changes. Mode-0 outgoing DEV requests
for its own slot arm monitor user 8; outgoing AP receive-channel requests arm
user 0. Cross-band changes set preparation and hold flags before writing the
user's field. The deadline is threshold plus 16 with byte wrap.

An auxiliary context and payload bit `0x20` can select subchannel 1 after the
role lock is released, even if role-specific work was ignored. AP suppresses an
incoming change for any nonzero ignore-once value; DEV suppresses only positive
values. Immediate successful completion occurs only if no monitor completion
is pending; it need not prove that physical retuning happened.

Monitor state spans ten users, with shared success/unlock counters. The third
counted success clears a pending completion and posts `0x109`; a failed post
does not restore it. Periodic misses wrap as bytes. Deadline equality can post
failure before the loss callback. Cleanup preserves unrelated callback changes.
Reset requests occur every ten misses after 19 for AP or 50 for DEV, subject to
suppression; mode 0 resets globally and other modes reset a user.

The historical runtime integrates normal mode-0 user-8 loss monitoring and
deferred recovery. Delayed polling advances once, without invented catch-up
ticks, and RX reset waits for owned DMA views to be released. Hopping and
channel-completion workers remained unintegrated in that stage. Broad helper
comparisons do not establish those runtime capabilities.

## Frequency resolution and tuning

The low channel byte indexes the catalog; all remaining bits specify subchannel.
Frequencies are masked to 24 bits. Invalid indices resolve to zero. Nonzero valid
subchannels apply configured offset and spacing. Bandwidth codes 0–5 select
`1250 << code` kHz; other codes yield zero width. Arithmetic preserves unsigned
wrap, and the separate subchannel-bandwidth setting is not used in this formula.

Tuning writes five PLL bytes, then band/path fields. TX can apply a band offset
and refresh power under a mutex; narrowing and state changes precede the power
callback, whose failure does not undo them. RX users below 11 receive frequency
and gain hints, with user 10 using the default. Ordered gain rows select the
first threshold strictly above frequency, or the final row. Metadata updates
follow these effects.

The 11,184 tuning and 2,328 resolution comparisons used synthetic PLL words and
intercepted writes. They establish ordering and arithmetic, not physical support
for every supplied frequency. Runtime integration in this receiver stage was
limited to measurement RX11. See [power-control findings](../firmware/ar8030-power-control.md)
and the [validation limits](../experiments/open8030-validation.md).
