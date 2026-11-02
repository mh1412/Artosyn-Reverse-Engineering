# G11 CRSF contact, electrical tests and reset failures

[Experiments](README.md) · [G11](../devices/skydroid-g11.md) · [Driver contract](../firmware/g11-crsf-transport.md)

## Result and scope

The 2026-10-26 bench investigation located the custom MCU's CRSF output on the
G11 **RX/PPM contact**. An optional active-drive build subsequently improved
the unloaded signal to about 3.16 V with a 47 ns rising edge. Connecting the
Nano still produced faults and resets, with no received bytes or valid replies
in the recorded monitor window. Receiver delivery and failsafe behavior were
not established.

This record concerns the GD32 controller and external module connection. It
does not qualify the separate internal Artosyn radio. All measurements below
are recorded experiments on one handset, not newly performed tests.

## Contact identification and configuration

The enabled model used external CRSF at 400 kbaud with 16 channels. Revision 2
changed two bytes of the 7,991-byte model blob: offset 5502 changed `0 → 5`
(module type), and offset 5504 changed `0 → 8` (stored channel-count field).
These are that model format's offsets, not a general-purpose patch recipe.
The startup throttle check had to pass before RF output was enabled.

Running-target reads verified PB7 AF0, open-drain output with pull-up, USART0
swap/half-duplex configuration and an advancing RC frame counter. PB7 is the
USART0 RX alternate-function pin; the swap setting permits its use for TX.
The scope remained armed without acquiring on the adjacent TX contact. Moving
only its tip to RX/PPM produced the waveform, and the operator reported valid
frames. This supersedes earlier contact names based on assumed probe placement.

No independent UART/CRC decode was completed from that capture. The result
locates the output; it does not establish direct PCB continuity, absence of
an intervening buffer or inverter, or a working return path. The UI's
`module_sent` counter counts management traffic, so zero there does not negate
an advancing MCU RC-frame counter.

## Electrical comparisons

The initial scope connection used a direct cable, 1 MΩ DC input and 1× scaling.
At 400 kbaud one bit lasts 2.5 µs. The early rising edges therefore occupied
more than one bit interval.

| Recorded condition | High/top | Rise | Interpretation |
| --- | --- | --- | --- |
| Initial RX/PPM capture; Nano still on adjacent TX | 1.16 V | 4.8 µs | Output found, electrical compatibility unqualified |
| Nano moved to RX/PPM | 1.20 V | 4.17 µs | No MCU replies observed |
| Nano supplied at 10 V, open-drain signal | 1.16 V | 5.08 µs | Correcting module supply did not resolve low signal |
| Nano signal removed, custom open drain | 3.12 V | Slow edge remained | Attachment affected high level; automatic multi-bit rise measurement was unsuitable for a single-edge claim |
| Original stock 0.5 PPM, Nano signal disconnected | 3.28 V | 67 ns | Same contact can carry a strong, fast push-pull signal |
| Custom active drive, Nano signal disconnected | 3.16 V | 47 ns | Unloaded drive improvement verified; fall time 50 ns |

The Nano initially used a 5 V USB supply; the experiment notes recorded its
specified supply as 7–13 V and changed it to 10 V. That change was necessary
for the comparison but did not establish supply voltage as the sole cause.
An opposing module input pull-down was one hypothesis based on inspected
ExpressLRS 3.3 source. The installed Nano firmware and actual input circuit
were not identified, so this remains a hypothesis.

Removing the scope initially coincided with a failed startup throttle check
and disabled output. That interval cannot establish whether scope loading
affected active CRSF. Later steady windows with advancing counters and no
reset likewise do not substitute for a controlled power transition.

## Controlled module power cycle

Before the decisive test, sticky reset history was acknowledged and read back
as `0x00000003`. Only the Nano's 10 V supply was cycled; the handset supply
was unchanged. One off/on pair took approximately 12 seconds between tool
results, including orchestration latency, rather than the requested two-second
sleep alone. No MCU halt or reset command was submitted.

The 74.995-second capture contained 918 samples:

1. Boot 131 entered control fault `6` (`RC_IO`), with `crsf_fault = 1` and RF
   disabled, around the module power-on transition.
2. Approximately 1.15 seconds later, the newly set `0x20000000` FWDGT flag
   identified a watchdog reset. Boot 132 started.
3. Boot 132 emitted 204 RC frames, faulted again and reset after another
   approximately 1.15 seconds.
4. Boot 133 remained running through the rest of the capture, with no Nano
   replies. Linux's boot ID changed and its UI returned.

This establishes the escalation **CRSF driver fault → RC_IO → watchdog reset**
and the accompanying Linux reboot in this test. The driver's failure path
cleared peripheral state, so the snapshots cannot distinguish a receive error
(parity, framing, noise or overrun) from a TX deadline failure. Sequential
register reads are not atomic; a sampled boot value of zero represents
transient initialization. The experiment did not identify a mixer/deadline
fault or prove a particular electrical trigger.

## Stock PPM control

The original 0.5 MCU image was restored after two matching backups of all
128 KiB of custom flash. Option bytes were unchanged. An erase command timed
out, but subsequent reads found the whole flash blank with the controller idle
and CPU halted; the erase was not repeated. The stock image was programmed and
verified before reset.

Stock application execution used `VTOR = 0x08002000`. Its recovered power-state
events brought Linux back. A bounded stock UART transaction read coach mode
`S` with switch 7, wrote mode `P` while preserving switch 7, received `OK`, and
verified the readback. The custom UI was disabled while the stock protocol
owned the UART.

PB7 was a push-pull GPIO output without a pull-up/down. With the Nano signal
disconnected, the same RX/PPM contact produced the fast 3.28 V PPM waveform.
At 5 ms/div, the capture showed PPM groups. The scope's approximately 500 Hz
automatic frequency was a channel interval, not a complete PPM-frame rate.
This control demonstrates available drive strength at the contact; stock PPM
with a Nano load and bidirectional CRSF were not tested by it.

## Optional active-drive build

The experimental driver enables push-pull only during transmission on the
verified swapped USART0/PB7 route. It restores open drain before returning to
receive mode, including fault, abort and shutdown paths. The default driver
retains open drain. See the [transport contract](../firmware/g11-crsf-transport.md)
for transition ordering and timing limits.

The installed ELF had 45,024 bytes of fast code/data and 20,740 bytes of cold
content. Adding the driver exhausted the former fast-region margin; only final
shutdown was moved into cold code, after acquisition stops. The complete flash
image was verified before reset, and the model returned at revision 2 with
Linux reporting zero local mixer cycles.

The unloaded scope result confirmed sharper edges. Attempts to download a
complete raw waveform timed out, so this is not an independent CRC-decoding
result. Reconnecting the Nano produced the following sequence in a 179.979-second,
2,206-sample monitor:

- Boot 135 entered `RC_IO`/CRSF fault, then reset about 1.15 seconds later.
- Boot 136 emitted 28 RC frames before the same fault/reset sequence.
- Boot 137 resumed advancing TX counters, but the receive-byte and valid-reply
  maxima remained zero.

Reset flags were already sticky in this run; it is consistent with the earlier
established watchdog path but does not independently identify reset cause.
The connected scope showed approximately 0 V with 80 mV peak-to-peak, and PB7
read digitally low while output was subsequently inhibited. The Nano supply
was 10 V. These observations do not diagnose a damaged module or prove polarity.

## Final disconnect and unresolved measurement

With CRSF inhibited, the operator disconnected the Nano signal. GPIO input
changed from `0x7952` to `0x79d2`, making PB7 digitally high, while the pin
remained AF0/open drain with pull-up and USART reception enabled. A fresh AUTO
scope acquisition still showed about 0 V and 80 mV peak-to-peak.

The probe endpoint therefore remained uncertain: it could have been on the
removed module-side wire. A digital high is not a measured 3.3 V level. This
last trace cannot locate the voltage drop or prove line contention. The record
ends with RF inhibited on boot 137 and no new actively driven test after
disconnect confirmation.

## Artifact identities and evidence limits

| Artifact | SHA-256 |
| --- | --- |
| Original stock 0.5 flash image | `58ad76e89664dd9a7d687f08dafa55c1bc81af1e44ccc674650fcfb4eb41883e` |
| Custom flash backup before stock comparison | `e2a23d2a8cd3bd49e614981a5ec56e6f6df879c0bd61834d15eb25ae450a41b3` |
| Active-drive ELF | `5077ea21ab3f92d7564823cc10ed3b126da6eff5904eb78d150ac51f07ec6f16` |
| Verified active-drive complete flash target before reset | `53a41a9d3b0d61cadccc9234b889cebe7c53d27fda60e906895fa14974f660dd` |

This synthesis uses the dated contact-verification, power-cycle, stock-PPM,
active-drive and disconnect reports, the associated installation report and
development-log entries. It preserves reported scope measurements; raw captures
have not been independently decoded for this wiki. Open work includes verified
probe placement, loaded voltage and turnaround measurements, retained UART
fault causes, module firmware identification, valid replies and receiver tests.
