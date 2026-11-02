# G11 CRSF half-duplex transport

[Firmware](README.md) · [G11](../devices/skydroid-g11.md) · [Electrical experiments](../experiments/g11-crsf-electrical.md)

## Ownership and configuration

The custom GD32 driver owns USART0 for a single-wire, non-inverted 8N1 CRSF
connection. USART1 remains the Linux management link. USART0 must not also be
owned by the full-duplex driver. GPIO routing, pin electrical configuration,
NVIC setup and clock selection are board responsibilities.

The installed G11 route uses PB7 AF0 with TX/RX swap and half-duplex mode;
its output was physically located on RX/PPM on 2026-10-26. The stock firmware
also assigns USART0 to PA9 AF1, so conflicting alternate-function routes must
be removed before reuse. Swapping UART pins does not establish signal inversion
or bidirectional board continuity.

Configuration accepts the actual USART clock and a requested 400 kbaud–2 Mbaud
rate. It rejects dividers below 16 and quantization error above 1%. Divider
acceptance is not proof that the attached module supports that rate. The
TIMER1 microsecond timebase must already be running; the driver does not select
the RCU USART clock source.

## One frame at a time

The driver starts receiving, with TX disabled. It accepts one validated CRSF
frame of 4–64 bytes and has no TX queue. Frame copying remains interruptible;
readiness is checked again after the copy so newly arriving RX data can block
the send.

During transmission RX is disabled. TBE advances the data bytes, clearing stale
TC before each write. Only the final hardware TC releases the transmitter and
returns to RX. An empty data register alone does not mean the last stop bit has
left the pin.

After TC, another send requires all of these conditions:

- A reservation of 68 character times has elapsed: a maximum 64-byte reply plus
  four characters of margin.
- At least two quiet character times have elapsed since the latest RX byte.
- The hardware reports no receive activity or unread byte.
- The software receive ring is empty.

A late or long reply can extend the wait. These checks cannot prevent contention
with a peer transmitting outside the expected response opportunity.

## Timing budget at 400 kbaud

An 8N1 character takes 25 µs. The GD32 adds an initial idle character before
the transmitted frame. The driver also allows 200 µs for TX interrupt delays.

| Frame | Frame plus initial idle | RX reservation | Including TX-delay budget |
| --- | --- | --- | --- |
| 26-byte channel frame | 675 µs | 1,700 µs | 2,575 µs |
| Maximum 64-byte frame | 1,625 µs | 1,700 µs | 3,525 µs |

These are scheduling calculations, not measured end-to-end latency. The early
driver notes proposed 5 ms as an initial external period; later autonomous
bench counters used a 4 ms cadence. The portable session's 2 ms lower bound
does not make 400 kbaud operation at that period feasible. The board adapter
obtains the configured driver's minimum period and rejects shorter OPEN
requests. Actual interrupt latency, line timing and peer behavior still matter.

## Receive buffering and fault behavior

The receive ring holds 128 bytes with a microsecond IRQ-service timestamp for
each. These timestamps preserve service order and age across foreground delays
and timer wrap; they are not hardware edge timestamps. The runtime must drain
the ring promptly and use a bounded frame-gap policy. Inspected ExpressLRS 3.3
code can split a large reply across handset intervals, so clearing every
partial frame at each reply-window boundary would lose valid traffic. That
source version was not established as the installed Nano firmware.

RX error, ring overflow or TX timeout latches a fault and disables USART
outputs and interrupts. Overflow invalidates access to all older buffered
bytes. The TX deadline includes initial idle, frame duration and the 200 µs
delay allowance. IRQ, read and readiness calls check it; a stalled CPU still
requires an independent watchdog. Only explicit configuration clears a fault.

`abort()` resets an active transfer and resumes listening with a fresh wait
window. When already listening it is idempotent, and it does not clear a fault.
`shutdown()` disables outputs and invalidates the driver before handoff. Neither
operation implements flash updates or native bootloader power/metadata entry.

The 2026-10-26 module power-cycle test exposed the installed integration's
failure escalation: a driver fault became control `RC_IO`, stopped watchdog
feeding, reset the MCU and also rebooted Linux. Error state was cleared too
early to distinguish RX errors from TX timeout in the retained snapshots.
See the [experiment record](../experiments/g11-crsf-electrical.md#controlled-module-power-cycle).

## Optional active drive

The `G11_CRSF_ACTIVE_DRIVE` profile is restricted to the verified swapped
USART0/PB7 route. Startup and reception remain open drain with pull-up. After
enabling TEN, the driver selects push-pull before loading the frame. On final
TC it restores open drain **before** disabling TEN and enabling REN. Fault,
abort, reconfiguration and shutdown likewise release active drive before
disabling or resetting the peripheral; other GPIO output-type bits are preserved.

The default build remains open drain. The experimental profile improved
unloaded edge speed, but connected-module operation still failed. Modeled
transition ordering and a sharp unloaded edge do not establish physical
release timing, safe peer turnaround or a working receive phase.

## Validation boundaries

The original register-model campaign reported 133,806 ARM calls across 17
scenario groups and a maximum 90-instruction masked span. The later dated
active-drive report records 133,127 calls across 16 groups, a 95-instruction
masked span and a 748-byte state object; its default-driver comparison records
133,817 calls across 17 groups and a 90-instruction span. These counts describe
different builds, and modeled instruction spans are not measured interrupt
latencies.

Cases covered rate rounding, swapped routing, single-frame ownership, TC-only
release, stale TC, receive/quiet windows, RX arrival during copying, ring/timer
wrap, overflow, errors, stalled TX, abort and shutdown. The test ELF lacked
startup/vector code and was not a flashable board image. A separate combined
board-loop simulation exercised the real mixer/configuration code with modeled
peripheral boundaries; it did not qualify physical DMA, IRQs, flash programming,
watchdog timing or RF delivery.

The physical evidence establishes contact location, unloaded waveform
improvement and the fault/reset sequence. It leaves module replies, receiver
delivery and failsafe behavior open.
