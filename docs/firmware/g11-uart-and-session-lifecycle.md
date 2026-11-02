# G11 internal UART, ownership and session recovery

[Firmware](README.md) · [G11](../devices/skydroid-g11.md) · [Custom wire protocol](../protocols/g11-custom-control.md)

## Hardware route and driver

The custom internal link uses MCU USART1 on PA2/PA3 and Linux `ttyS1` through
`ms_uart`, device-tree node `/soc/uart1@1F221200`. Linux `ttyS0` is the separate
kernel console; USART0 is the separate MCU CRSF interface.

Downloaded stock 0.8 code ties native update handling to USART1: IRQ
`0x080089ac` reads RDATA `0x40004424` into the ring at `0x200014c8`, main
`0x08005cca` consumes it, and parser `0x08005e0e` dispatches A1/A5. Query and
BOOT responses call the helper writing TDATA `0x40004428`. Stock pin setup
selects PA2/PA3 AF1. This establishes that image's software route, separately
from PCB continuity or installed-image verification.

The custom full-duplex driver configures 8N1 with 16× oversampling from a
supplied clock, rejecting divider error above 1%. At 48 MHz, dividers are 417
for 115200 and 52 for 921600. This excludes oscillator tolerance. It enables
and resets only the selected USART, preserving unrelated clocks. Invalid
configuration leaves the existing instance unchanged; an initialized instance
cannot migrate between USARTs.

One frame of at most 82 bytes is copied outside the interrupt-masked section,
then fault state is rechecked before starting its first byte. There is no
extra TX queue. Each IRQ services at most one received and one transmitted
byte. TBE fills TX, while final TC completes the frame; stale TC is cleared
on every write. `idle()` remains false until final TC is serviced.

The 128-byte RX ring uses wrapping 16-bit counters. Overflow, parity, framing,
noise or overrun latches failure, disables the UART and makes reads return −1,
including when earlier clean bytes remain. Only reconfiguration clears failure.
The historical runtime then stopped RC/recovered; a driver flag alone cannot
stop a separate RC peripheral without timely runtime service. Later autonomous
management recovery must be assessed under its own control-ownership policy.

The per-instance ARM state is 228 bytes. One foreground owner and the matching
IRQ own the APIs; critical sections preserve PRIMASK. Register-model tests
observed 48/44 masked instructions for one-/82-byte sends, demonstrating that
masking did not grow with packet copy length. These are not elapsed-time or
interrupt-latency measurements.

## Pin setup and startup boundary

Internal-link setup temporarily disconnects PA2/PA3 to pulled-up inputs, sets
their latches high atomically, then applies AF1, push-pull and the 10 MHz speed
setting: TX without a pull, RX with pull-up. It runs after peripheral
quiescence in privileged Thread mode with interrupts masked. Locked pins or
failed readback abort startup. Other GPIO/clock fields, SWD, external CRSF and
power pins are preserved. A partial failure does not permit retry into control.

The historical adapter connected USART1, USART0, ADC/DMA, TIMER1 and watchdog,
requiring coherent initial acquisition and persistent nonce allocation inside
100 ms. A nonce write finishing after the deadline could not advertise a late
startup. IRQ28/27 dispatch internal-link/CRSF handlers, with equal priority and
global interrupts enabled only after preparation. Initial poll expectations
of 100 µs were implementation requirements, not measured physical bounds.
Early fixture ELFs without production board startup were not installable
firmware; later installation records must be used for hardware claims.

## Console-induced losses and mitigation

Overruns and decoder rejects occurred at both 921600 and 460800 baud. Routing
IRQ34 to CPU1 separated it from display/video IRQs on CPU0, but short 100 Hz
input tests still added two overruns in 183.7 seconds at 921600 and three in
183.6 seconds at 460800. Those counts do not establish a significant baud-rate
difference.

A related board-family driver uses a 32-byte FIFO and an IRQ-saving spinlock
during console output in groups of up to 16 characters. The running kernel
exposed matching console-function names, but the inspected source was not an
exact installed-binary match. Frequent Wi-Fi logs and console level 7 motivated
a controlled test emitting 100 known INFO lines on CPU1 without opening the
MCU UART.

| 460800-baud burst | Overrun increase | Input result |
| --- | --- | --- |
| Console level 7, first | 6 | One decoder reject; remained live |
| Console level 4, first | 0 | Continued inputs, no new reject |
| Console level 7, second | 7 | Two rejects; session expired |
| Console level 4, second | 0 | Already expired: invalid reliability comparison |

This reproduces one failure path, not the cause of every earlier loss. Recovery
used maintenance and RUN-existing without an application write or Linux reboot.

The installed development guard kept IRQ34 on CPU1 and console level 1, while
all messages remained in the kernel ring for `dmesg`. It preserved unrelated
printk fields and restored original settings on explicit stop only if they
still matched its own settings. It changed no baud, kernel image, FIFO or MCU
pin. Emergency console output and other kernel stalls remained possible.

Two filtered-console tests used 1,000 INFO messages and 40 menu open/close
cycles:

| Baud | Duration | Additional inputs | Additional overruns/rejects |
| --- | --- | --- | --- |
| 460800 | 300.01 s | 29,679 | 0 / 0 |
| 921600 | 300.08 s | 29,684 | 0 / 0 |

Both stayed live with RF off and no live decoded video workload. A later
90.22-second reference-scan run still added one overrun and one reject with
console level 1. Its cause was unresolved; the chronological ADC change does
not establish causation. The guard is a mitigation, not a loss-free or hard
real-time guarantee.

## Exclusive process ownership

The Linux backend uses one nonblocking UART descriptor, flock/TIOCEXCL,
visible process-descriptor checks, bounded reads/writes and kernel output-queue
checks. It saves/restores termios. SIGSTOP does not release a vendor owner's
descriptor. A failed second opener must not clear the first owner's exclusive
mode. Process scans can miss inaccessible owners or privileged races, so
supervision remains necessary.

The custom resident-session runner sequences maintenance client → uploader →
UI. It never opens the UART itself. It holds a shared private lock whose file
must be owned by the current user, mode-private, regular, single-link and not
a symlink. Cleanup never replaces/unlinks the inode. The inherited lock survives
child exec, so helpers must remain foreground and retain it; unrelated programs
that ignore the lock are not excluded by that mechanism.

For a running control app, a headless maintenance-only client performs discovery
and maintenance without starting a mixer, video or Wi-Fi workers. The runner
requires a fresh private `G11M` notice **and** successful client exit before
starting the uploader. With a new image, offline validation precedes maintenance
and the uploader validates again against the loader before erase.

The uploader requires its own 115200 loader HELLO/OPEN. Without an image it
requests RUN-existing. A failed or lost RUN ACK prevents UI startup; ACK alone
does not prove application execution. A new UI must send a private `G11R`
notice only after a live session and fresh mapped input, with nonzero count and
shared-host monotonic timestamp less than 100 ms old. Default readiness timeout
is 30 seconds, configurable 100–60,000 ms. This one-time check neither arms RF
nor continuously monitors health.

Arguments go directly to exec without shell evaluation. Conflicting ownership
or command/capture modes are rejected. HUP/INT/TERM terminates the child group,
waits up to five seconds, then kills/reaps if needed. Parent-death handling
kills the direct child. No phase proceeds after shutdown, and there is no
automatic retry loop. Forced kill can bypass termios restoration.

## Hardware startup and explicit reconnect

Early target smoke tests used `/bin/true` and synthetic readiness fixtures;
their RUN-acknowledged logs were not MCU evidence. The later October 24 physical
sequence used the actual maintenance client, uploaded 12,976 application bytes,
received RUN ACK and started real EdgeTX with fresh mapped input. Only the UI
held `ttyS1`; Linux boot ID remained unchanged. Control used 460800 while the
loader used 115200, with RF forced off. One startup reject was observed. Later
921600 deployment and boot-service installation superseded the earlier
manual-only status.

The historical Controller link page offered explicit Reconnect inputs after
expiry, creating a fresh token on the same UART without MCU reset, flash writes
or Linux reboot. It cleared pending output/telemetry/feedback. RF stayed off
until explicit startup checks; input-only deployments never enabled it.

With UI `92f13bef…`, two 300 ms SIGSTOP/SIGCONT cycles expired the input-only
session while a known 720p25 HEVC stream decoded. Reconnect restored fresh
controls each time. SWD showed generations `1 → 2 → 3`, unchanged MCU boot
nonce 65 and `VTOR = 0x08004000`. UI process and Linux boot were unchanged.
This validates the UI action on hardware; it does not prove physical finger
touch, external CRSF or receiver failsafe.

A separate transition audit sampled steady input, a 300 ms stall, reconnect,
maintenance, a three-second resident wait and RUN-existing. Overrun count
stayed 156 throughout; break count changed `42 → 44` at maintenance. This
distinguishes breaks from overruns for that handoff, without explaining the
earlier rise `108 → 156`. A later watchdog reset rebooted Linux and reset its
counters; comparing them across boot as a decrease would be invalid.

## First-fault diagnostics

The later `link-stats` diagnostic retained the first fault snapshot from the
most recently failed session across explicit reconnect. It included input age,
current/largest poll interval, time since poll, token, sample identity, rejects
and pending userspace/kernel UART bytes before flushing. One bounded failure
log was emitted after expiry; healthy polling did not query queue occupancy.

Long poll gaps support a scheduling-delay explanation; old input with short
poll intervals instead suggests data was not accepted in time. Neither alone
identifies the cause, and queued bytes need not be valid frames. Unavailable
queue sizes are −1. Unsigned 32-bit microsecond subtraction handles ordinary
wrap, but cannot represent a full-wrap gap (about 71.6 minutes). Largest sampled
interval is not a worst-case bound. Process restart clears these diagnostics.

These session-expiry mechanics belong to the earlier Linux-mixer design.
The later [autonomous MCU](../soc/g11-gd32-acquisition.md) continues mixing
independently of Linux management loss and must not inherit a Linux heartbeat
requirement merely because both designs use the same physical UART.

The later [autonomous debugger-interference experiment](../experiments/g11-video-and-debugger.md#debugger-interference-isolated)
reproduced UART recoveries during live executable reads, with clean adjacent
idle windows. It supplies a separate measured cause of some later disturbances;
it does not retroactively explain every earlier overrun.
