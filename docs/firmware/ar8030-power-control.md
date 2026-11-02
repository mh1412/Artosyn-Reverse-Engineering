# AR8030 power-control reconstruction

[Firmware](README.md) · [Board configuration](ar8030-configuration.md)

These findings describe matched 18.21.10 stock behavior and historical receiver
stages 188–190. Register gains, configuration levels and reported cache values
are not calibrated RF output power. Later P1 provisioning has a separate
[calibration record](p1-calibration-and-settings.md).

The earlier [startup-gain experiment](../experiments/ar8030-uplink-and-login.md#gain-correction-produced-matching-ap-login-records)
first obtained matching AP login records after correcting effective TX0 fields
to the stock acquisition profile. That bounded comparison precedes the general
power-policy reconstruction here and did not measure physical output power.

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
`-3`. Native validation rejects unsafe users, missing calibration, unsupported
configuration and absent callbacks before writes. Caller serialization is
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

## Initial runtime application

The loader integration supplies resolved rows, path offsets, clock/role settings
and CSMA adjustment. Runtime owns its calibration copy and initializes user 0;
other user caches are not inferred from registers or copied from user 0.

Before RF writes, the historical receiver integration requires the supported
DEV/high-clock/closed-loop profile and checks that the requested initial level
reproduces the four planned gain values. Missing rows or disagreement reject
preparation. Saved host configuration retains calibration for restart, reinit
and recovery. Diagnostic plans carry precomputed gains and clear the ordinary
power context instead of accidentally inheriting it.

The matched connected-state comparison exposed a policy difference:

| Case | Gain/amplitude pairs | Meaning in that calibration |
| --- | --- | --- |
| Stock user 0 | `(34,39)` on both paths | Row 14 |
| Accepted native user 0 | `(23,78)` and `(22,78)` | Initial row 25 |
| Rejected dynamic candidate | `(37,30)` on both paths | Row 11 |

The stock backend requested setter `0x02000008` with bytes `00 0e`, user 0 and
level 14. The accepted native image rejected that setter and stayed at startup
level 25. The same native gains existed before helper integration. This supports
the missing-policy explanation, without independently proving RF-power accuracy.

## Manual command contracts

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

## Failed dynamic integration

Stage 190 experimentally integrated the commands and all-user initialization.
The normal backend selected level 11 before acquisition. All 33 sampled states
in the 30-second trial remained disconnected/searching, and HDMI showed the
disconnected UI. The candidate was rejected and its runtime integration removed.

The earlier accepted image also failed acquisition when only its temporary
initial-power configuration changed from 25 to 11. This reproduced failure
without the new dynamic setter, but did not locate its RF cause. A full reboot
and restoration of stage 189 at level 25 restored the camera scene and 33 linked
samples in the follow-up 30-second check. Only compared helpers/tests were
retained from the failed integration.

Dynamic setters, tuning callbacks, other users, valid factory records and
measured power were not established by this historical receiver trial. Later
board-specific work must carry its own firmware and calibration provenance.
