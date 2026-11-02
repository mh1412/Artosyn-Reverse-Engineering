# G11 stock MCU images, trainer timing and feedback commands

[Firmware](README.md) · [G11](../devices/skydroid-g11.md)

## Artifact identity

The official 0.8 update was obtained on October 22 through the manifest URL
embedded in the installed HAL. A bounded A1 version query on October 23 later
identified the connected MCU as **0.5 / G11**. Its complete reply was:

```text
66656e6779696e676469616e7a693aa0050005473131df
```

The 0.5 download came from the derived versioned server filename, not a current
manifest entry. Downloaded applications, installed full-flash reads and option
bytes are different artifacts:

| Artifact | Size | SHA-256 |
| --- | --- | --- |
| Official 0.5 application | 28,186 bytes | `c87fedfa8b496db7415f94e5710041a607d11f0aa01e1b0e5a65a71d66b9b560` |
| Official 0.8 application | 28,731 bytes | `eac4a6da934416a96af9aae3df04a3e177a8a3c69cb0e2b3b6ba6988d2e05d3e` |
| Installed stock full flash, two matching reads | 128 KiB | `58ad76e89664dd9a7d687f08dafa55c1bc81af1e44ccc674650fcfb4eb41883e` |
| Installed option-byte backup, two matching reads | Separate region | `c0b942fbb9fe967ec0e7b675e080d48c930fc5fe3fde70f6dd6f9646fdffc0d3` |

The newer release adds a virtual-channel interface for channels 12–16,
corresponding to SDK command `0xd1`. Its presence in the download does not
establish that feature in installed 0.5. Later SWD comparison matched the
installed application to the archived 0.5 bytes.

## Binary layout and identification

The 0.8 image is directly disassemblable Thumb code with a Cortex-M vector
table. It starts at inferred `0x08002000`, initial MSP `0x20000cf8`, reset
vector `0x08008f0d` (code at `0x08008f0c`, file offset `0x6f0c`). Startup calls
`0x08008e84` and `0x08008ec8`, enables CP10/CP11 through CPACR and executes
FPSCR setup before main `0x08005220`.

IRQ/peripheral matches, GPIOB at `0x48000400` and executed FPU setup initially
suggested a GD32F3x0 Cortex-M4F family. They did not independently identify a
package or memory suffix. The photograph was read as GD32F330C8T6, but later
density and restoring physical tests established 128 KiB flash/16 KiB SRAM on
this unit. The suffix reading remains unresolved. See
[memory qualification](g11-mcu-memory-and-models.md); old 64/8 capacity arguments
must not be treated as the final measurement.

The initial GPIO helper matched between 0.5 and 0.8 in offline execution:
29 setup calls, 245 MMIO writes, 556 instructions and 20 literal words under
two initial register patterns. This did not compare every later power/clock
action. GPIO A/B fields are cleared selectively, but C/F helpers OR into
existing mode/type/speed/pull values. PC13–15 and PF6–7 can retain inherited
state; a software jump does not supply reset defaults.

## Captured trainer slave stream

The October 23 stationary capture used 8 MHz sampling for one second. D0 was
active, D1–D7 high. It contained 99 complete **44-byte** frames, all with valid
XOR checksums, and one partial boundary frame. The electrical capture decoded
as non-inverted **115200 baud, 8N2**, approximately 100 frames/s.

| Frame offset | Content |
| --- | --- |
| 0–8 | ASCII `SKYDROID:` |
| 9 | `0xb1` |
| 10 | `0x20`, 32-byte payload |
| 11–42 | Sixteen big-endian u16 channel fields |
| 43 | XOR of bytes 0–42 |

Median frame interval was 10.001875 ms; observed range 9.947125–10.108875 ms.
Byte-start spacing was 95.5–95.75 µs, consistent with eleven-bit characters.
The byte decoder used one stop bit, but both expected stop-bit centers were
separately checked high for all 4,396 decoded bytes. Payloads were stationary
and identical, so this capture alone did not establish channel movement or
physical input latency. Digital sampling did not measure voltage.

The operator reported probes on TX/RX, but exact contact assignment was not
resolved in that capture. Later RX/PPM scope identification is separate
[electrical evidence](../experiments/g11-crsf-electrical.md), not a reason to
retroactively claim its original probe placement was known.

## Trainer modes in downloaded 0.8

Command `0xb8` carries mode `P`/`M`/`S` and trainer switch H=7, B1=10 or B2=11.
The MCU generates output independently of the Linux display's polling loop.

In slave mode S, the packet builder near `0x08008690` fills sixteen fields in
the buffer at `0x20000024`, XORs bytes 0–42 and writes byte 43. TIMER1 at
`0x40000000` drives PB7 through ISR `0x08008a50`: start low, eight LSB-first
bits, two high stop intervals. A final preload of byte 44 is not a transmitted
45th byte. Reload 416 and prescaler zero are consistent with the firmware's
assumed 48 MHz clock; the physical capture supplies the baud evidence.

PPM mode P uses the timer at `0x40002000` and PB7, with ISR `0x08005744`.
Eighteen states produce eight channel intervals: nine fixed high phases of
500 counts, eight variable low phases from successive values at `0x200000a4`
plus 400 counts, and a 7,000-count sync low. A normal-operation state gates
output. These are timer counts from static analysis, not automatically
microseconds or a fixed PPM frame rate. The later stock PPM scope comparison
verified a waveform on RX/PPM but did not establish every channel's timing.

Master mode M validates the same 44-byte serial frame and decodes its first
four fields. Depending on trainer-switch state it substitutes or combines
them with local controls and clamps results. Exact switch positions and
timeout behavior remained unqualified. Master/slave communication is not PPM
merely because PPM is another menu mode.

The stock Linux dual-format reader later converted trainer values into display
channel values using the traced 0.5 formula. See
[stock input formats and mapping](../protocols/g11-stock-inputs.md).

## Beep/vibration API limits

The SDK defines two consecutive bytes, `beep` then `vibrate`, each documented
as none/short/long. Captured HALs export set/get functions omitted from the
public header's prototypes. There is no per-motor selector, strength or explicit
duration. The setter sends command `0xcc`, which the synchronous transport
special-cases to return without waiting for an actuator reply, synthesizing
OK if needed. Success therefore establishes transmission, not vibration.

In 0.8 the parser validates a 20-byte command and sets pending flag
`0x20001cf5`. A halfword-aligned PC-relative-load scan found only that parser
reference; no main-loop consumer was found. This suggests an unimplemented
command in that image, while indirect/address-arithmetic references and other
versions remain possible. Installed app imports and inspected SDK call sites
did not establish use of the API.

Built-in button/calibration events drive buzzer patterns. A low-battery mode
also pulses PB10 under PB11/repeat gating, but its load was not identified.
No standard Linux vibrator/force-feedback endpoint was found. These facts do
not establish two independently controllable motors. Later custom
[feedback/audio work](g11-feedback-and-audio.md) deliberately leaves PB10 alone.

## Historical alternatives and selected direction

Early proposals included replacing the stock trainer packet with CRSF, decoding
the proprietary stream in modified Nano firmware, or using a Backpack network
control service. None of those proposals established an implemented stock-Nano
bridge. A 44-byte 8N2 packet takes about 4.20 ms at 115200; faster RF cannot
create more than the observed approximately 100 fresh source frames/s.

The proposed Nano parser would have had to remain input-only and suppress
handset-bound replies because stock S-mode keeps its output driven. Ordinary
main-TX Wi-Fi update mode in the inspected ELRS source stopped RF operation;
Backpack live telemetry was a distinct possible route, not an existing generic
configuration API. These alternatives were superseded by the selected custom
G11 MCU/UI with stock Nano firmware and bidirectional CRSF. Later implementation
and unresolved connected-module results are documented separately.

Keeping stock/custom binaries in Linux storage never implied two executable
MCU slots or automatic rollback. The early nominal-64-KiB calculation left
118 bytes too little for two 28,731-byte apps after an 8 KiB reservation, before
alignment/data. That was a historical design constraint, not proof of the later
measured capacity or a bootloader slot mechanism. The actual
[update and recovery history](g11-mcu-update-history.md) separates those stages.
