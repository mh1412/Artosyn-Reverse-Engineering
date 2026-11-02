# AR8030 power-control reconstruction

[Firmware](README.md) · [Board configuration](ar8030-configuration.md)

These findings describe matched 18.21.10 stock behavior, trials 145–158, the
subsequent offline checkpoint and historical receiver stages 188–190. Register
gains, configuration levels and reported cache values are not calibrated RF
output power. Later P1 provisioning has a separate
[calibration record](p1-calibration-and-settings.md).

## Calibration and cache order

The recovered application path performs open-loop field writes, updates a shared
current-level byte, performs closed-loop writes, then updates the addressed
user's cached level. The shared byte aliases user 0's cached level. Applying
user 8 or 11 therefore changes user 0's reported level without necessarily
changing user 0's hardware gains. Stock hardware checks also reproduced this
reporting effect when setting user 5.

Closed paths independently clamp signed path-plus-user offsets. Open paths use
unadjusted analog rows. Amplitude scaling depends on the clock; DEV user-8 CSMA
adjustment can reduce gain with zero amplitude. Invalid byte-sized levels return
`-3`. Caller serialization is
required; the application helper does not acquire the original mutex itself.

The power and band-offset comparison covered 63,060 original-instruction cases
with synthetic calibration/state. It checked ordered field writes and state
at each write, including the cache alias. MMIO remained intercepted, so this
does not measure emitted RF or establish dynamic host-policy integration.

## Band offsets

Offsets are signed bytes. Values outside −6 through +6 do nothing. If both paths
are closed, the recovered paired-step table selects gains. Otherwise the first
nibble is 12 and the second is 12 plus offset clamped to −3 through +3. The first
field is written before the second.

Users 0–9 have paired per-user fields; user 10 has dedicated fields and user 11
has none. This operation does not change calibration offsets or cached levels.
The TX tuner narrows an offset before invoking power refresh. A failed refresh
does not undo that stored offset change.

### Local policy and peer policy are separate

| Command | Early reconstructed behavior |
| --- | --- |
| `0x01000009` | Three bytes: peer-adaptation enable, upper limit, lower limit |
| `0x02000009`, selector 0/1 | Disable/enable peer adaptation and store limits from input bytes 1/2 |
| Same setter, selector 2 | Store byte 1 as all four local initial/reset levels; ignore byte 2 |
| Same setter, selector 3 | Store byte 1 as automatic/manual policy; nonzero selects automatic levels |

Selectors 2/3 do not immediately write gain fields or change reported/applied
levels. The original dispatcher stores the policy bytes directly. Original fallback
uses `pwr_init` default 28, overridden by `auto_init`; automatic/manual reset
levels default to that result.

Connected-slot message `0x305` resets power only when clearing a previously
connected bit; clearing the last bit resets all users to the selected disconnect
level. Repeated failed acquisition does not count as connected loss. The early
single-slot DEV implementation preserved policy across stop/start. Hardware
trial 157 verified selector 2 storage without immediate gain changes; trial 158
also verified selector 0 disable. Selector 1 enabled adaptation, selector 3 and
connected-loss resets retained offline/synthetic coverage in those trials.

### Incoming automatic-power selection

The checkpoint after trial 158 integrated incoming FCH command 8/subtype 6
offline. Wire CRC/length/TTL checks and native connection gating precede the
selector. The supported DEV mode-0 path requires local automatic policy,
the assigned slot and a primary request; it ignores secondary requests and
maps the selected request to user 0.

The payload carries an eight-bit level, four-bit direction, four-bit slot and
secondary flag. Level 255 requests a relative step: direction 1 increases,
direction 2 decreases, and other directions retain the current level. The
candidate must fit the configured local range. Explicit levels bypass that
range; an explicit increase already at or below the current level instead
advances one step when below the maximum. The current value is user 0's
**reported** byte, which can differ from its applied gains after another
user's setter. Calibration validation still precedes any application.

Local `pwr_range` defaults to 10/28 and is 25/28 in the matched JSON; it is
separate from peer-adaptation limits. Native parsing requires exactly two
byte-valued integers and preserves inverted ranges, in which an out-of-range
relative candidate leaves the current level unchanged. That selector matched
3,344 retained original-instruction cases with helpers intercepted; it was not
a hardware trial 159. Later trials 159/160 included local and peer-policy
handling but had no connected link or incoming update, so they did not establish
its hardware acceptance.

### Peer-policy synchronization and decision state

The subsequent subtype-8 callback updates global DEV peer-adaptation policy
without requiring login or local automatic power mode. Its slot nibble does not
filter the update. Primary and secondary enable bits are independent; five-bit
upper/lower limits propagate only to configured rows. Inactive rows, history,
hold timers, MCS thresholds and queued requests survive. This command changes
no local gain. Host selectors 0/1 instead carry full-byte limits and set both
primary enable and secondary flag bit 0, preserving higher auxiliary flags.
Retaining a secondary flag does not implement mode-1 adaptation.

The decision component tracks a sixteen-sample error bitmap/count, consecutive
errors capped at sixteen, smoothed SNR `(old + 3*sample) >> 2` and the lower of
the two gain values. MCS above 24 maps to zero for row selection. Error, streak,
SNR and gain factors are evaluated in order; an increase decision stops further
factor evaluation. Up/down hold timers and sample-spacing gates precede queueing.
The candidate uses byte arithmetic before clamping, including wrap. Peer level
zero substitutes the configured upper limit.

A queue attempt records the requested candidate, but updates its last-successful
sample only when the queue callback succeeds. The helper's nonzero result
therefore indicates an attempted request, not RF delivery or even guaranteed
queue acceptance. Incoming limits preserve the decision history rather than
resetting adaptation on every command.

The peer-policy callback matched 3,072 retained original cases, including all
1,024 limit pairs. The decision comparison retained 1,536 samples, including
285 requests and 81 queue-busy outcomes. Helpers were intercepted; trials
159/160 supplied no incoming-update hardware acceptance. ## Manual command contracts

| Command | Recovered meaning | Reply |
| --- | --- | --- |
| `0x01000007` | Power mode | One byte |
| `0x01000008` | Cached user level | Two bytes: user, level; invalid user yields level `0xff` |
| `0x02000008` | Direct level setter | Empty; invalid level returns `-3` |

The setter locks, applies calibration/cache changes, then unlocks. Native
malformed requests and unsafe setter users return `-22` before effects.
Comparison covered 12,550 command scenarios and 4,096 all-user initialization
scenarios, including getter observations after setters.

The original all-user loop can return mutex-release status after application
failure. The independent initializer preflights every user's calibration and
returns validation errors; 3,600 deliberate differences were recorded. They
must not be counted as exact error parity.
