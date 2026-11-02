# VR04 display pacing and diagnostic measurements

[Systems](README.md) · [Receiver latency experiments](../experiments/vr04-receiver-latency.md)

## Scope

This study covers the complete pacing, persistent-breadcrumb and burned-counter
implementations at MissingLynk revision
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`. It describes their current control
and measurement boundaries. Historical latency and clock-fit results remain
in the linked experiment page; no new panel, clock or camera test was performed.

## Current pacing loop

Pacing is enabled by default when the pixel-clock sysfs attribute can be opened
and read. `ML_PACE=0` disables it. An in-range numeric value can seed the hold
rate, while other strings do not generally mean off: the disable check requires
a leading `0` and a parsed value of zero. The initial rate is parsed without
independent validation against the panel's physical refresh.

The current sensor is **frame-ready to flip-event callback**, not merely
flip-submit to event. This includes time spent behind an already pending frame.
It uses software callback timing and inherits the
[display event/latch uncertainty](vr04-frame-ownership.md#display-mailbox-and-retirement).

| Controller setting | Current value or rule |
| --- | --- |
| Target wait | Starts at 6500 µs; adaptive range 6500–10,500 µs |
| Target adjustment | +1200 µs after an attributed miss, otherwise −120 µs, when previously settled |
| Phase deadband | Strictly inside target ±1200 µs |
| Retained sensor samples | First 64 positive waits per nominal one-second interval |
| Minimum sample count | 30 retained waits |
| Low-tail sample | Sorted element at index `n/20`; approximately the fifth percentile |
| Proportional action | Speed-up only, capped at 0.6% of the hold rate |
| Hold-rate drift correction | After idle proportional intervals; at most ±8000 Hz per tick |
| Command range | 141.1–155 MHz on the steady-video path |
| Hold-rate floor | Rate read at process startup |

A low-count interval requests the startup rate while preserving the converged
hold rate for later use. Because interval length is not measured, a delayed
main loop is not normalized into a true samples-per-second rate or drift rate.
More than 64 waits increments the incoming count but does not replace the first
64 retained samples with more recent ones.

The loop attributes a long flip interval to insufficient margin when the
previous wait was below target plus 2 ms; otherwise it calls it a source gap.
That is a heuristic using two observed intervals, not an independent causal
measurement. It raises the target once per eligible tick with any attributed
miss, rather than once for every missed refresh.

The proportional error is calculated before that tick adapts the target.
Consequently the command can use the old target while the log prints the new
one. An offline example widened 6500 to 7700 µs but still computed its command
from the 6500 µs error. The source comments' monotone convergence and margin
equilibrium descriptions are design arguments, not unconditional guarantees
under sampling delay, actuator error or changing input cadence.

## Playback, writes and shutdown

During playback the timer discards wait samples and makes no clock adjustment.
It does not clear all miss/gap counters, the previous phase observation or the
settled-state history. Flip callbacks can continue collecting that history.
Return to RF is therefore not a complete controller reset.

A full-length sysfs write updates the cached commanded rate; failed or short
writes leave that cache unchanged. Other controller state can already have
advanced. Debug output prints the desired command, and there is no readback
after each write to establish the actual rate or completion of the driver's
ramp. A write accepted by sysfs is not an optical refresh measurement.

Normal teardown attempts to restore the startup rate and then closes the
attribute. That attempt has no readback/settling verification and does not run
after every abrupt process death. A new process can therefore inherit a rate
left by an earlier instance. The current loop and its startup floor must not
be substituted for the older fixed-margin controller when interpreting dated
bench results.

## Forced-phase hang investigation

`ML_PHASE_FORCE` accepts a positive target below 16,000 µs. It delays the tile-0
copy path to try to place a vblank at that offset after submission. The active
implementation anchors to the most recent hardware-vblank timestamp obtained
through DRM, superseding its older event-relative description. Period estimates
accept 15–18 ms samples and use a smoothed average; the timing calculation
assumes the vblank and monotonic timestamps share a compatible clock domain.

The requested sleep is below one estimated period, but scheduler delay can
overshoot it. Feedback averages the phase error and clamps bias to ±300 µs.
Hits mean the reported offset lies within ±100 µs of the target.

These counters require careful interpretation:

- Completion feedback runs after a successful direct tile-0 DMA copy, not every
  displayed frame or every fallback copy. Its `phase_flips` count names those
  completion calls, not independently observed page flips.
- A changed vblank sequence increments the “inside” count, but the function
  does not check that the timestamp lies between copy start and completion.
  An offline case with a timestamp 100 µs before submission still counted as
  inside and contributed a negative offset.
- If several vblanks pass, the reader observes the latest one rather than an
  event-by-event trace. A low hit rate does not establish that the intended
  narrow timing window was exercised.

Forced phase adds delay to the experiment. The latency summary labels that
mode, but its figures must still be kept separate from normal-operation latency.

## Persistent breadcrumbs

When `ML_PMSG` is present, the process writes phase, PTS in milliseconds,
monotonic microseconds and, for tile operations, the channel to `/dev/pmsg0`.
The intended retained-RAM path depends on the kernel's pstore/ramoops setup and
the reset preserving that memory; it is not general power-loss persistence.

| Marker | Instrumented boundary |
| --- | --- |
| `B` / `b` | Entry/return of the broader tile-copy region |
| `D` | Successful direct DMA submission has returned |
| `K` | Seam-band capture call has returned |
| `I` / `S` | Display submission entered/returned successfully |
| `E` | Accepted flip callback for a pending frame |

An unmatched `B` can cover DMA, staging, CPU copy, cache flushing or band work.
It does not identify one ioctl by itself. Marker writes are best-effort with
unchecked return values; the device is opened without nonblocking mode. Ignoring
a write result neither guarantees a complete trace nor proves the write cannot
delay the display path. An absent return marker can also be a lost record or
ring wrap, so the last marker does not alone prove which call hung. Retention
duration depends on line lengths, enabled phases and kernel record overhead.

## Burned counter and the actual measurement interval

The counter samples receiver monotonic milliseconds at composite assembly.
It renders five digits modulo 100,000, plus a sweep that resets every 100 ms.
The sweep spans 800 pixels, giving eight rendered pixels per millisecond before
camera scaling. The outer frame occupies 876×306 luma pixels at `(522,168)`;
the inner box begins at row 180. Luma ink is 140 against a 16-level background,
with neutral chroma and a red border.

The same modified composite feeds display and recording, avoiding a separately
timed HUD drawing. In the proposed feedback experiment, an air camera films
the receiver's panel, and a later recorded composite contains both an older
filmed counter and the newer locally burned one. Their subtraction needs no
clock synchronization between air and ground.

It does, however, measure **old composite timestamp to new composite timestamp
through the display/camera feedback loop**. It includes the old frame's display
queue/scanout and camera/air/receive path, and ends before the new composite's
own final presentation. Equating that interval with another camera-to-screen
definition requires its display timing assumptions to be checked. Merely drawing
identical pixels into display and recording does not remove those endpoints.

Exposure can combine multiple displayed digit values into a plausible wrong
seven-segment number. The sweep was designed to make that ambiguity easier to
detect, but its “later edge” interpretation assumes a particular superposition
model. Long exposures, compression, stale frames and sweep resets require
validation; the renderer alone proves no universal one-frame error bound.
The 100-second numeric rollover also requires an assumed interval range when
subtracting values. Approximate row-scanout delays in the comments are not a
substitute for the unresolved panel timing calibration.

## Offline verification

Exact pacing/phase functions were tested with fake clock writes and vblank
replies. Checks reproduced the old-target command, failed-write cache behavior,
retained playback history, first-64 sampling and negative “inside” interval.
The full unchanged counter body, with host dependency declarations and a fake
clock, passed disabled-output, bounding-rectangle, sweep-width and 100-second
rollover checks. Both probes passed warnings-as-errors and sanitizer checks.

These checks do not establish physical lock, real ioctl timing, persistent
trace retention, hang reproduction or a numerical optical latency result.
