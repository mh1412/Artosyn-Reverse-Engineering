# G11 MCU trim actions, Instant Trim and Flight Reset

[Firmware](README.md) · [Autonomous engine](g11-autonomous-engine.md)

These operations modify MCU working state without moving control ownership to
Linux. Their deployed software must be distinguished from the physical G11 input
map: the tested configuration has no assigned trim contacts and no automatically
assigned Instant Trim button.

## Trim contacts

Revision 9 extracts the pinned EdgeTX `Key` state machine and `checkTrims()` from
commit `64c7221001332dba962c90abdf76ca9c89f8840b`, with section hashes checking
upstream drift. Hardware debounce uses four samples despite simulator headers.
The engine samples coherent raw contacts at its 10 ms boundary and applies
events after mixing/functions; changed trims affect the following mix.

Behavior includes accelerating repeat, center pause, normal/extended limits,
release/repress at the normal limit with extended trims, throttle steps,
stick-mode mapping, inherited/additive/independent/disabled trims and three-position
handling. It retains one pending event and ascending key-scan order, including
that policy's handling of simultaneous keys. Activation clears debounce/repeat/
pause/killed-key/event state; reconnect does not.

Positive trim-source GVar functions reuse the mapped buttons to adjust bounded
inherited GVars instead of trims. Both scaled/raw modes take this upstream branch.
The validator requires at least one mapped contact for the logical trim after
stick-mode mapping. Negative trim sources retain numeric-source semantics.
Increment settings are limited to −2…2; malformed −4/−3/3 are refused before
shift arithmetic. Dynamic output-limit checks still apply.

Migration from revision 8 to 9 is allowed only when all twelve trim contacts
are unassigned. Earlier firmware could observe contacts without adjusting trims;
silently enabling new behavior would change outputs. Such models require explicit
new configuration. This is a closed compatibility transition, not general ABI
conversion. Trim observations/checkpoints use the existing runtime paths; this
does not introduce Linux trim injection or reassign G11 switches.

## Instant Trim

Revision 10 adds the pinned `instantTrim()` to model/global functions. Two input
evaluations exclude trainer data and compare stick contributions with a
sticks-disabled reference. The operation skips throttle, honors trim-source
exclusion, refuses conflicting contributions from one stick to multiple inputs,
and applies upstream deadband/limits. It changes the working overlay without
editing immutable model/radio bytes.

A half-second startup floor precedes function execution. Supported repeats are
`!1x`, `1x`, and 1–60 seconds. A held one-shot must release/reactivate to fire
again; `!1x` suppresses a startup-held function. The startup floor completes once
per activation, and a separate fired bit distinguishes a real zero timestamp
from an unused timer across 32-bit 10 ms clock wrap. Linux reconnect resets
neither list's function timing.

Revision 9 refused Instant Trim, so the pinned identity-only 9→10 migration
cannot enable an assignment that was valid previously. Both MCU slots and Linux
journals migrate together, preserving configuration and working values. Revision-10
schema is `0x4f1154c66333b468`; earlier trim revision 9 used `0xd7268735e2c3669b`.

Ten-model ARM/native comparisons covered 19,250 normal steps and 8,000 warm-change
steps; combined tests checked 595 emitted channel frames, Linux loss, inhibit and
wrap. Physical boot 115 retained 499.82–500.15 Hz RF-off mixing with no assigned
Instant Trim action. One handset freshness lapse recovered in the next sample;
the deployment did not establish its cause or physical Instant Trim execution.

## Flight Reset

Revision 13 (`0xeb04e1ae06c8dfff`) adds the control portion of upstream Flight
Reset. It resets non-manual timers to configured starts, clears telemetry validity/
extrema and logical-switch state, clears the mixer first-run flag, and restarts
half-second function silence. Manual-reset timers, GVars and trims remain intact.
Receipt cursors remain, preventing cached telemetry from undoing the reset.

Configured once/once-except-startup/1–60-second repeat actions defer changes until
current mix/timer/trim work completes. Their own repeat bookkeeping survives the
reset, so a held one-shot cannot reset each cycle. Model/global functions share
the dispatcher; UART commands invoke the same helper between cycles.

Flight Reset leaves an already-running channel stream enabled. Startup warnings
become observations instead of blocking GUI checks. Repeated resets update one
dialog; UI restart baselines the current count without replaying old warnings.
There is no throttle-trace accumulator in this MCU profile to reset.

Explicit UI reset requires fresh matching MCU status and idle model/calibration/
shutdown jobs. Linux startup/local model-loading resets remain local. Lost ACKs
are reported as uncertain; exact packet retry is deduplicated and reconnect does
not resubmit commands. The [reset wire contract](../protocols/g11-autonomous-management.md#timers-sources-sensors-and-resets)
also supports timer, telemetry, current-mode trim and individual sensor reset.

## Physical reset evidence

Revision 13 was deployed as boot 121 with matching UI and migrated journals.
The physical active bench model had disabled timers and zero trims/GVars/logical
state. Opening/canceling Reset session left the reset count at zero; one explicit
confirmation advanced it to one without changing UI PID 750, MCU boot, model
generation or RF-off policy. Subsequent status still reported one. The 334-byte
working overlay matched SHA-256
`b0c198071c3e56fdc7e3de8eb4330627ca4ec020f11575964466a4dd252a45a5`
before and after. An earlier navigation attempt only woke the backlight.

This establishes one idle physical reset and cancellation behavior. Nonzero
timers, repeat functions, telemetry history and active-radio continuity use
combined ARM/native/UI tests with simulated peripherals. It does not establish
receiver output, physical trim wiring, acoustic feedback or worst-case timing.
Later [feedback work](g11-feedback-and-audio.md) separately implements local trim
tones and presentation; those capabilities should not be inferred from these
earlier trim-action tests.
