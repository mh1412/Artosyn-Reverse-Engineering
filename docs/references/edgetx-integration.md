# EdgeTX integration reference for G11 research

[References](README.md) · [G11 Linux UI](../systems/g11-linux-ui.md)

The retained EdgeTX documentation at
`64c7221001332dba962c90abdf76ca9c89f8840b` explains several assumptions behind
the G11 experiments. It is a historical software reference, not a G11 hardware
specification or a statement about the latest upstream release. This review
covers the text; embedded protocol waveform images were not reanalyzed.

## Controls, names and physical identity

The input architecture distinguishes physical control names, canonical logical
names and translated/custom UI labels. Calibration refers to physical controls;
input and mixer references use canonical functions such as Rud/Ele/Thr/Ail.
Stick mode connects these layers. Additional controls include potentiometers,
sliders, joystick axes and switches.

The APIs preserve that separation: physical-name lookup, canonical-name lookup
and user-label access are different operations. Hardware definitions use a JSON
model, validation and Jinja2-generated includes, with separate legacy naming
metadata. A display label alone therefore cannot identify an ADC channel,
physical direction or calibration record. The
[G11 stock-input mapping](../protocols/g11-stock-inputs.md) retains its actual
slot mapping and physical observation limits.

## Mixer synchronization belongs to an owner

The upstream mixer task triggers ADC conversion, computes channels and sends
them to the module. For supported synchronized protocols, a timer schedules
each cycle with adjustable frequency and phase. The module supplies feedback;
the regulation loop lives in the module and adjusts the next mixer cycle to
reduce unwanted sampling/transport timing differences. The FrSky heartbeat
mechanism instead triggers the mixer and resets its safety timer.

This explains the distinction between regular channel output and a
module-synchronized loop. A nominal 500 Hz calculation rate alone does not prove
phase alignment with RF transmission or measured acquisition-to-receiver latency.
The later G11 architecture assigns acquisition, mixing and timing correction to
the MCU. Linux is an observer/editor, and Lua configuration replies must not
reactivate Linux mixer timing. See
[module configuration ownership](../protocols/g11-module-configuration.md) and
[CRSF transport](../firmware/g11-crsf-transport.md).

The retained external-module overview lists several different serial formats
and labels Crossfire as 400000 baud, inverted, with radio-dependent reductions.
That terse reference is not a measurement of G11 RX/PPM polarity, electrical
drive or baud rate. The [G11 electrical experiments](../experiments/g11-crsf-electrical.md)
remain authoritative for the studied connector and preserve the connected-load
failure separately from successful unloaded waveforms.

## Generated YAML metadata is part of the ABI

The YAML generator uses `libclang` to inspect `ModelData`, `RadioData` and related
structures, producing tables that map named fields to bit offsets and lengths.
These generated sources are committed; the ordinary build does not regenerate
them. Changing a structure or target configuration can therefore leave a
compiling application with stale persistence metadata.

The retained generator guide explicitly identifies implicit bitfield padding
as a limitation. Its prescribed representation makes padding explicit and marks
it `SKIP`, retaining its bit count without serializing it. Custom read/write
functions handle special formats and compatibility; selectors choose union
members; explicit enum annotations provide symbolic values even where the C++
field uses an integer type. These mechanisms require target-appropriate layouts,
not merely a valid YAML document.

This supplies the context for the G11 native top-bar mismatch: an older six-slot
serializer did not match the native eight-slot structure. The
[recorded fix](../systems/g11-linux-ui.md#native-rendering-and-settings)
regenerated matching descriptors and restored prototype fields. The reference
does not establish compatibility of arbitrary desktop and embedded layouts or
automatic migration of every existing model.

## Simulator and UI evidence

The simulator interface exposes lifecycle, channel/output/logical-switch
observations and callbacks for analogs, keys, trims and display. Its declared
stop operation is synchronous despite an asynchronous intent. These interfaces
explain how native fixtures can exercise real mixer/UI code without qualifying
physical acquisition or module delivery.

Likewise, generated font tables and translated function ordering are distinct
from YAML layout generation. The retained build notes describe a font generator
that can warn and skip an unreadable translation, so successful script completion
alone is not proof that every language's glyphs were regenerated. The G11's
actual framebuffer, persistence and controller observations remain separate
checks in the [UI study](../systems/g11-linux-ui.md).
