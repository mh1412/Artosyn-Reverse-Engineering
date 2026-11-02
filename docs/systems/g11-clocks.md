# G11 Linux time, RTC and independent MCU clock

[Systems](README.md) · [G11](../devices/skydroid-g11.md)

The system has distinct Linux wall time, Linux monotonic deadlines, an RTC and
an MCU software clock. Correcting a date must not suspend control observation,
and a displayed valid date does not prove retained or accurate physical time.

## Linux date editor and the backwards-time defect

The native Radio Settings editor sets Linux `CLOCK_REALTIME` and saves UTC to
`/dev/rtc0`, preserving the configured Linux timezone. The observed device used
UTC; this change adds neither network synchronization nor a timezone editor.
Dates must round-trip through calendar conversion, including DST rules, and fit
the editor's 2023–2037 range and target signed 32-bit time.

If Linux time changes but RTC write/readback fails, the UI reports that partial
result. It does not roll back or silently retry. Polling/startup never writes the
RTC. `system_valid` means representable editor time; `rtc_valid` means readable
representable RTC date, not synchronization. Civil display seconds and actual
Unix epoch fields have different meanings.

The first physical UI build saved time successfully, but moving time backwards
stalled worker observation. The target GCC 8 condition-variable implementation
used wall time for `wait_for`. Explicit `CLOCK_MONOTONIC` condition waits fixed
the defect while the date display continued to follow wall-clock changes.
Existing stale observations had recovered without a restart; that recovery
alone did not make the original wait implementation correct.

The corrected UI SHA-256 was
`e8f9ee4a34d601b349b2fb1109fc2e1eff3cc2c6a9161a06c7e83572d76947c4`.
With MCU boot 116 unchanged and RF off, six post-correction observations had
control ages 0–28 ms and input ages 5–36 ms. Time survived UI replacement/restart;
this test did not remove power or reboot Linux to qualify RTC retention.

## MCU holdover and correction

The MCU software UTC clock advances from its local microsecond counter in the
foreground loop. It keeps seconds, fraction, fixed local offset and update
revision. Linux seeds it and requests explicit corrections; no recurring Linux
time stream is required. Model stop/activation, UART reconnect and UI exit
preserve it. MCU reset clears it: the clock is not persisted in flash.

UTC is bounded to 2023–2037 and local offset to whole minutes from −12 through
+14 hours. An observed gap of at least half the microsecond-counter period or
date overflow invalidates time. The watchdog must prevent an unobserved whole
counter gap. Holdover follows the MCU oscillator; it is neither calibrated nor
a battery-backed MCU RTC. DST/offset changes require explicit correction.

Service bit 2 advertises model operations READ 18 and SET 19. READ supplies a
one-use challenge and current revision. SET supplies both, UTC seconds/fraction
and signed local offset. The challenge expires after 250 ms and OPEN invalidates
it. The MCU checks output/shutdown policy at execution: correction is refused
during enabled CRSF or pending shutdown, consuming the challenge. Exact retry
returns the cached result without a second correction.

The UART worker samples Linux time after receiving the challenge. It checks
caller identities and does not replay uncertain corrections after reconnect.
The UI seeds a never-set clock once per observed boot when Linux time is
representable and output is off. It does not overwrite a valid clock on UI
restart. Date edits update Linux/RTC first, then request MCU correction, reporting
any partial failure. These checks provide ordering, not authentication or an
external precision-time reference.

## Time as an MCU model source

Capability revision 11 feeds the MCU clock into the engine before each mix.
Pinned EdgeTX uses local minutes since midnight, including valid zero at midnight;
signed comparisons retain minute units. Unavailable time is zero with validity
false, so input fallbacks and conditional/GVar handling can distinguish it from
midnight.

Dependency validation covers direct mixes, input lines, numeric weights/offsets/
curves, logical operands and model/global functions. A time-dependent model may
mix while waiting, but startup output stays pending until time is valid. Once
that engine has observed valid time, losing it causes terminal control failure
instead of false time-dependent output. Models without time references remain
independent. Warm activation clears engine observations, not the board clock.

Model startup bit 32 means clock required. Native time Value/Gauge widgets use
independently fresh MCU observations and show unavailable values as `--`; they do
not substitute Linux RTC data. Revision 11 requires matched cache/UI/journal
schema migration. Earlier statements rejecting all time sources predate this
integration.

## Physical deployment sequence

| October 26 stage | Recorded result |
| --- | --- |
| Boot 117, MCU `4cc60516…`, UI `f03ba97c…` | Independent clock seeded, physical minute edits in both directions, revision 3 after restoration |
| Boot 118, MCU `6e11de1f…`, UI `95c78f3b…` | Revision-11 engine time source, both model slots/mirror/selection and Linux journals migrated with configured/working values preserved |

Full flash and option readback passed at both stages. Boot 117 completed 1,505
mixes during 3.008748 seconds of UI suspension, about 500.21 Hz. Initial analysis
incorrectly assumed atomic SWD word reads; 133 µs skew prompted a repeat with
an explicit sub-millisecond tolerance. The passing read had 89 µs skew. This is
a measurement-method correction, not proof of oscillator accuracy.

Boot 118's physical engine `g_rtcTime` followed clock edits and advanced during
a 3.010664-second UI pause with 1,505 mixes, about 499.89 Hz. Whole-second date
edits and delayed observation initially left a roughly two-second reference
offset; a hidden follow-up report bounded it between −2.708 and −1.637 seconds.
Linux/RTC and MCU were explicitly resynchronized, finishing at revision 4 with
fresh controls. The clean date-page framebuffer was recorded in the deployment
report; it was not re-rendered for this documentation review.

These finite RF-off tests establish deployment, edit behavior and short holdover.
Oscillator accuracy, power-removal retention, heavy-model timing and actual
time-dependent receiver behavior remain unqualified. Combined ARM tests cover
time-source rules, midnight, counter wrap, simulated multi-day operation and
Linux loss with simulated peripherals.
