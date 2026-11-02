# G11 stock decoder, fallback picture and display lifecycle

[Systems](README.md) · [Video ingestion](g11-stock-video-ingest.md) · [Video control](../protocols/g11-stock-video-control.md)

Scope: the decoder, fallback-picture and display implementation in vendor SDK
revision `b3917ed3c3a0eef0afc6548ba6a00cffc61d2293`, including their headers,
selected panel table and VideoPlayer Makefile. Full source review establishes
the behavior below; it does not establish equivalence to an installed player
or a new physical display result.

## Decoder configuration and error reporting

The shared decoder type is initialized to H.265 and this source has no setter
that changes it. Normal channel creation explicitly uses H.265, frame input
mode, normal DPB mode and these values:

| Setting | Value |
| --- | --- |
| Configured input dimensions | 1920×1080 |
| Reference-frame count | 5 |
| Stream-buffer size | 2,073,600 bytes |
| Priority | 0 |
| Device low-latency-disable field | `g_bframe` |
| Output dimensions | Minimum of requested video/window dimensions, rounded down to multiples of 32 |
| Decoder output-port depth | User depth 0, total depth 3 |
| Main decoder/display binding rate arguments | -1 and 60 |
| PIP binding rate arguments | 30 and 30 |

A 1920×1080 request/window therefore produces a 1920×1056 decoder output
request. The binding's rate arguments are not measured display cadence or
camera-to-screen latency. Preview mode is requested for channel 0.

The four-entry configuration array and loops depend on callers to enforce the
channel-count domain. PIP uses decoder channel `VChan_num` and display channel
16; it is not always decoder channel 4 as some comments suggest.

Device initialization and preview-mode returns are ignored. Create, start,
output configuration and binding failures are logged through `STCHECKRESULT`,
which does not abort or return an error. `sstar_vdec_init` still returns 0.
A host fixture forced every vendor API to fail and reproduced continued
create → start → output-configuration → bind calls and a zero return.

## Reset and unbind are different operations

| Function | Actual operation | Reported result |
| --- | --- | --- |
| `sstar_vdec_reset` | Unbind channel 0, flush it, stop it, destroy it | 0 regardless of these API results |
| `sstar_vdec_is_init` | Call the destructive reset, then initialize/bind one decoder channel | Always -1 |
| `sstar_module_unbind` | Unbind decoder 0 from display 0, then disable display input port 0 | Success despite logged failures |
| `sstar_vdec_deInit` | Unbind, stop and destroy configured channels and optional PIP channel, then deinitialize the decoder device | 0 despite logged failures |

The reset's “keep channel alive” message contradicts its active stop/destroy
calls. The initialization check's old proc-file test is commented out. It is
therefore a recovery attempt with side effects, not a read-only readiness query.
The fixture verified both sequences with mocked calls.

The [RTSP reuse review](g11-stock-video-ingest.md#rtsp-pause-resume-and-url-changes)
can now be stated more precisely: consumer exit both unbinds and disables the
display input port. A matching-parameter decoder reuse path has no explicit
rebind/enable operation. This is a source-level gap, not a reproduced panel
failure or proof of behavior in another player build.

## What the fallback-picture thread does

The picture worker opens `/customer/logo/start.h265` and repeatedly reads NALs
through the shared file parser. `/customer/logo/no_stream.h265` is defined,
but the file-switching block is commented out. The activity-based guard is
also commented out: this worker does not independently wait for stream loss
before sending its initial file.

`sstar_picture_switch` only sets a flag to 1. The worker then sleeps one second
per iteration and sends nothing further. There is no active assignment that
clears the flag. Calling “switch” again does not toggle back, open the no-stream
file or restart fallback playback. The monitor calls this function on mode
transitions, including picture mode; the picture-mode proc command only toggles
the decoder's output-buffer control.

The sender uses decoder channel 0, PTS 0 and a 20 ms send timeout. Every NAL is
marked end-of-frame, including parameter NALs. Certain four-byte-start-code
header patterns trigger a **100 ms** sleep; the nearby 30-fps comment does not
describe that delay. Three-byte start codes do not take that pacing branch.
The nine-byte `00 00 00 01 68 ... 00` rewrite to eight bytes is present here too.

On send failure, the worker tries to rewind by the submitted length, sleeps
30 ms and calls `sstar_vdec_is_init`, which destroys and rebuilds the decoder.
That return is ignored. Rewind failure is unchecked, and rewritten lengths or
EOF rewind behavior can make this different from retrying an identical parser
operation. Other producers are not quiesced by this recovery function.

The optional slice-drop branch logs a drop and increments its counter, but
does not skip the subsequent send. Its first-slice/time bookkeeping is not a
validated picture-boundary parser. It also examines fixed header offsets
without checking that the NAL contains them.

Normal cleanup frees the file and buffers, but allocation/open failures can
return before cleanup. Initialization ignores the thread-creation result;
deinitialization sets the exit flag and joins. Initialization does not reset
the exit or switch flags for reuse. Thus a mode ACK, a pause, and a restarted
fallback worker are distinct states.

## LCD geometry and the panel table

The LCD initializer calls `disp_port_init(1)` regardless of its requested
port count. Decoder initialization can still loop over a larger count, leaving
additional window records at their zero-initialized dimensions. A fixture
reproduced a second decoder output request of 0×0 in that state. The source's
multi-port and PIP support is not a demonstrated general multi-view pipeline.

Input source dimensions are rounded down to multiples of 32, while destination
window dimensions retain their requested values. The layer-size helper uses
the panel width/height directly only for rotation 180; other rotation values,
including none, swap them. Later RTSP port configuration instead reads the
panel's active width/height and uses them without that 32-pixel rounding. These
are separate configuration paths, not one universal displayed geometry.

The selected `CSOT045` table supplies:

| Property | Source value |
| --- | --- |
| Active dimensions | 1920×1080 |
| Horizontal sync/back/front porch | 2 / 44 / 20 |
| Vertical sync/back/front porch | 2 / 7 / 37 |
| Horizontal/vertical totals | 1986 / 1126 |
| Nominal `FPS` macro | 60 |
| DCLK field expression | `1126 * 1986 * 60 / 1048576`, integer result 127 |
| Display frame-rate expression | `127 * 1000000 / (1986 * 1126)`, integer result 56 |
| MIPI DSI configuration structure | Empty initializer, therefore zero-initialized |

The two clock calculations use different divisors. The derived 56 is the value
assigned by this source, not a measurement of panel refresh or proof that the
driver programs that exact physical rate. The table name likewise does not
independently identify the fitted panel. The empty DSI configuration is passed
to the panel API; this source alone does not explain inherited bootloader/driver
configuration or actual lane timing.

Compiling this table against the supplied panel datatype header confirms those
field assignments. It also places the values 160/255/80 in PWM period-low,
period-high and duty fields. The older initializer comments label them as
initial/maximum/minimum dimming values; those comments do not define the actual
structure or establish calibrated brightness limits.

The Makefile defaults HDMI support to 0 and the application entry selects LCD.
The conditional HDMI code requests 1080p60 by default, with a separate disabled
4K option. Its presence does not establish an HDMI output on this board.

## Screen commands and incomplete display teardown

Each screen command calls the panel API with a two-byte buffer `[0x7f, value]`.
This records the API input, not independently decoded DSI traffic on the wire.

| Operation | Command-value order | Explicit sleeps |
| --- | --- | --- |
| Screen off | `30`, `52`, `00`, `28`, `10` | 200 ms after `28`; 100 ms after `10` |
| Screen on | `11`, `29` | 150 ms after `11`; 50 ms after `29` |

The off sequence differs from the shorter sequence and timings in its comment.
A failed panel call stops the remaining sequence and propagates its error;
earlier successful commands are not rolled back. A fixture failing the third
off command returned that error before any sleep or GPIO shell command.

After successful screen-off, `sstar_disp_Deinit` runs four shell commands to
export GPIOs 48 and 85 and set their directions to output. It does not write
their values or check shell status. The input-port/layer/device disable and
panel deinitialization code is inside `#if 0`. This function therefore does
not perform the full display teardown its name/comment suggests. It also
does not close the graphics service opened during initialization.

The entry point calls this screen-off/GPIO routine before stopping the video
workers, then later deinitializes the decoder and system service. The ordering
does not prove that all users are stopped before screen-off. The
[control handler](../protocols/g11-stock-video-control.md#what-the-source-actually-applies)
leaves its screen-on call commented out, even though a screen-on helper exists.

## Verification and remaining boundaries

Offline fixtures compiled the unchanged decoder implementation and selected
display/lifecycle functions with substitute host types and mocked vendor APIs,
shell execution and sleeps. AddressSanitizer/UndefinedBehaviorSanitizer runs
passed for failure masking, output rounding, destructive reset/readiness
sequences, unbind/disable, screen command ordering, early failure and ignored
GPIO-command failures. Panel arithmetic was checked from the actual macros,
and a separate host compilation with the supplied datatype header verified
the populated dimensions, clock/PWM fields and empty DSI lane/command settings.

No hardware calls, GPIO writes or shell commands were executed by the fixtures.
They do not test a full SDK build, target ABI, concurrent worker recovery,
panel timing or visible fallback restoration. The
[physical restoration limit](../experiments/g11-video-and-debugger.md#restoration-and-capture-limits)
remains: the recorded experiment did not establish restoration of the original
fallback picture, and archived source behavior remains separate evidence.
