# G11 autonomous inputs, EdgeTX engine and configuration

[Firmware](README.md) · [G11](../devices/skydroid-g11.md)

The autonomous implementation runs acquisition, input calibration, EdgeTX mixing
and channel scheduling on the GD32 MCU. Linux owns model editing and presentation.
Reconnecting Linux does not restart the engine or supply a control clock. Earlier
component notes describe offline fixtures; later combined firmware and RF-off
bench results establish deployment, within the limits recorded below.

## Confirmed physical inputs

Movement tests on October 25 corrected the provisional exchange of LV and P1:

| Physical control | MCU input | Logical position |
| --- | --- | --- |
| Left stick horizontal | ADC0 | LH |
| Left stick vertical | ADC1 | LV |
| Right stick vertical | ADC6 | RV |
| Right stick horizontal | ADC7 | RH |
| Left knob | ADC5 | P1 |
| Right knob | ADC4 | P3 |
| Left shoulder | PC14 / PC15 | SA, three positions |
| Right shoulder | PB2 / PB12 | SB, three positions |
| Back button 1 | PF6 | SC |
| Back button 2 | PB4 | SD |
| Return-to-home button | PA8 | SF |

The three buttons report pressed = −1 and released = +1. The tested stick mode
was 2. This verifies assignment, not calibrated centers, spans or every control's
travel. Older saved P2 multiposition thresholds are not ordinary center/span
calibration.

The 52-byte physical map retains thirteen analog positions: LH/LV/RV/RH, P1/P2/P3,
two sliders and four extensions. Only the six control ADCs and seven known
control contacts are admitted. Battery, charge and power-button pins are separate
services. Missing optional analogs receive raw center values; absent switches
default up inside the adapter. The observation protocol separately marks absence.
Explicit contact aliases are allowed by this component; full model validation
must still establish source availability.

Configuration binds immutable model/radio/map data while control is stopped,
validates calibration and multiposition thresholds, and clears filter history and
readiness. Each coherent scan passes through upstream calibration, inversion,
jitter filtering and stick-mode mapping on the MCU. Invalid ADC values, missing
required validity bits or contradictory switch contacts clear readiness without
changing previous filtered values. The coordinator rejects that scan; preserved
old values cannot be treated as a new control sample.

The component build's `SIMU` and `G11_LINUX_UI` flags do not imply Linux execution:
the prepared ADC code explicitly retains real calibration under those flags.
Changing build flags requires checking that path again.

## Immutable configuration and mutable engine state

Pinned EdgeTX `ModelData` occupies 6,843 bytes and radio settings 1,064 bytes.
The engine borrows them from immutable flash instead of copying them into scarce
RAM. Mutable pointer types inherited from upstream do not authorize writes.
A 334-byte working overlay holds flight-mode trims/GVars and persistent sticky
state; timers have their own runtime state. The active model reads the lower
[flash mirror](g11-mcu-memory-and-models.md#disposable-active-mirror).

Warm activation must reset upstream hidden state too. Dedicated linker sections
collect mutable state from mixer, curves, GVars and logical switches. Reset
restores initialized data, including `lastFlightMode = 255`, as well as zeroed
storage. The measured reset set was two initialized bytes and 3,570 BSS bytes.
Link-time optimization would require rechecking that object/section isolation.
The combined engine explicitly initializes telemetry storage, timer/override
state and validated curve lookup; it does not depend on dynamic constructors or
repair malformed flash configuration in place.

Initialization failure stops the engine; it is not an atomic “keep old model
running” operation. Candidate validation therefore precedes activation. Mix
failure leaves the prior channel array unchanged but invalidates its use as a
fresh result. Unsupported bindings produce refusal rather than silently acting
as working zero-valued sources.

Supported control functions grew through capability revisions: channel overrides,
timer set/reset, trim reset, GVar constant/copy/rising-edge increment/decrement,
and admitted scaled/raw sources. Later revisions added trim-source GVars and
Instant Trim. Their ordering follows pinned upstream behavior, including the
quirk that an enabled model function list clears global overrides even if that
model list is empty. Validation checks reachable GVar ranges when they influence
output limits, not just initial values.

Configured logical switches execute in ascending index order. Sparse evaluation
skips unused entries while retaining all 64 switch slots and nine flight-mode
contexts. Full occupancy remains dense work. One no-logical-switch model improved
outer processing from 1,402 to 759 µs and periodic work from 450 to 15 µs; these
are model/build-specific observations, not a bound for arbitrary configurations.

## Scheduling and terminal faults

The scheduler supports 500 or 1,000 Hz; the documented deployed bench runs at
500 Hz. It preserves phase without replaying a backlog of missed cycles. Inputs
must be younger than 2 ms; CRSF eligibility uses the original acquisition stamp
and a 5 ms freshness boundary. MCU 10 ms ticks drive upstream timed functions;
Linux wall-clock changes cannot advance control.

Acquisition, engine, work-budget or radio-I/O failures latch a terminal control
fault. Model reinitialization cannot clear it; MCU reset is required. Linux
transport loss alone does not constitute such a fault. The combined board stops
servicing UART and feeding its watchdog on terminal failure, so a fresh fault
reply is not guaranteed before reset.

One pending management frame may use alternate external serial opportunities.
Unavailable transmit ownership for two channel periods after enable/last send
faults the channel path. Telemetry input is bounded to 64 bytes per poll, and
callback work must remain bounded: checking elapsed time afterward cannot undo
a blocking callback. See [CRSF transport](g11-crsf-transport.md) for actual line
timing and the unresolved connected-module failures.

## Configuration payload and validation

The model blob is exactly **7,991 bytes**:

| Component | Bytes |
| --- | ---: |
| Inner header | 32 |
| Pinned model | 6,843 |
| Pinned radio settings | 1,064 |
| Physical map | 52 |

It leaves 137 bytes in an 8,128-byte cache payload. The final odd byte uses
erased-byte padding. This is a packed little-endian ABI for a pinned build,
not a portable EdgeTX file format. Its schema includes upstream identity,
capability revision, layout, counts and source/switch numbering. For example,
revision 13 used schema `0xeb04e1ae06c8dfff` for the Flight Reset deployment;
that is a historical compatibility identity, not a timeless latest version.

The outer cache header owns model identity, revision, CRC and commit state.
The inner header repeats schema and exact structure lengths with reserved fields
zero. Decode validates everything before publishing borrowed pointers; failed
encode leaves its destination unchanged. CRC/schema checks provide integrity
and compatibility, not authentication.

Semantic checks cover available controls and calibration, source/switch numbering,
input/mix ordering, curves, delay/slow parameters, trim/GVar inheritance cycles,
output limits, logical-switch operands, timer modes and admitted functions.
Correct-CRC Lua or otherwise unsupported sources are still refused. Capability
support must be evaluated against the matching firmware/UI revision: later
telemetry/reset work supersedes some early blanket refusals. The documented
radio profile admits RF off or fixed external CRSF at 400 kbaud, sixteen channels
from CH1, legacy CH5 arming, zero custom centers and module-managed failsafe.
Validation does not establish receiver delivery or vehicle suitability.

## Evidence and limits

The input adapter matched an independent native upstream oracle for 10,128
samples across sixteen configurations and all four stick modes, with malformed
configuration/scan rejection and recovery. Engine fixtures compared 11,532 output
samples, 5,768 CRSF frames and 2,800 warm-up samples. Warm-reset testing covered
sixteen ordered transitions and 9,600 post-change samples; deliberately skipping
reset failed at sample 30. These are native/actual-ARM emulation comparisons.

Payload validation covered 113 targeted native/ARM cases plus 12,000 native and
250 ARM mutations. Fixture RAM and observed stack sizes are test-specific, not
production allocation or worst-case stack bounds. Physical deployment establishes
selected RF-off models near 500 Hz and Linux independence; it does not qualify
every accepted model's timing or a working connected receiver.

See [activation and persistent runtime](g11-model-activation-and-runtime.md) and
[Linux model management](../systems/g11-model-management.md) for how a validated
candidate becomes selected and survives reset.
