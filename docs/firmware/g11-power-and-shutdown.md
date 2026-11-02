# G11 power control, shutdown and reset recovery

[Firmware](README.md) · [G11](../devices/skydroid-g11.md)

## Evidence and implementation generations

Stock MCU tracing identifies several outputs involved in startup and shutdown.
Physical MCU resets have also rebooted Linux. The later autonomous firmware
implements a physical-button request, Linux save confirmation and MCU power-off
state machine. The complete deployed pair's physical off/wake gesture and rail
timing remain unqualified in the reviewed records.

Three designs must be distinguished: stock 0.5 behavior, the October 24 custom
resident-loader/Linux-mixer design, and the later direct-boot autonomous MCU.
The older coordinator's read-only remount procedure and exit code 42 are not
the later autonomous shutdown protocol.

## Stock signals and boot decisions

| Signal | Stock 0.5 behavior | What is established |
| --- | --- | --- |
| PB1 | Startup level depends on halfword `0x08001c08 == 0x4785`; raised at `0x080055fc`, cleared at `0x080056ac` | Strong power-hold candidate; rail/load not isolated |
| PB8 | Initialized high; raised at `0x080051e2` and `0x080055f2`, lowered at `0x080053be` | Operating-state output; load unidentified |
| PA10 | Initially low; high/low/high/low/high sequence at `0x08004632..0x08004672`; cleared at `0x080054d2` | Peripheral initialization pulse train; target and physical timing unknown |
| PB15, PA12, PA15, PB3, PB5 | Initialized low and cleared in the shutdown branch | Unidentified loads; cannot all be labeled power rails |
| PC13 | Active-low button | Software short/long-event source |

The 0.8 image has corresponding PB1 set/clear sites at `0x08005658` and
`0x08005706`; addresses and timing assumptions must retain their image scope.
The installed stock readback matched 0.5.

The stock bootloader's main at `0x08000fcc` enters its update loop if metadata
at `0x08001c04` equals `0xabcd`, or sampled PC13 and PB8 are both low. Otherwise
it disables interrupts, loads MSP from `0x08002000` and calls that image's reset
vector. No checksum check was observed in this branch. Eight emulated cases
used actual boot bytes with synthetic inputs and stubbed clock/delay functions;
they do not establish electrical timing.

## Observed resets and recovery

On October 24, SYSRESETREQ with unchanged stock flash returned to application
`VTOR = 0x08002000`, but its power state became zero and PB1 went low. Linux
and the display remained unavailable while SWD still worked. Injecting the
traced stock power-on RAM events (`0x20001ca8 = 100`, then `0x20001cb8 = 1`)
caused the stock branch to raise PB1, run initialization and enter state 2.
The UI returned and Linux had a new boot ID. This was a recovery experiment
against a known image, not a portable recovery recipe or proof that PB1 alone
controls Linux power.

A separate October 24 custom-firmware test injected its exact-build link-fault
byte in SRAM with RF disabled. Previously acknowledged reset history then
showed FWDGT (`0x20000000`). The last application-VTOR sample was 1.114 seconds
after injection, first watchdog indication 1.155 seconds, and resident VTOR
1.253 seconds. These include debugger and host sampling delays, not watchdog
bounds. Linux rebooted; Wi-Fi, UART policy and the UI boot service returned
automatically, running the existing application without an image write.

The entire 16,040-byte application and 8,132-byte aligned resident matched their
pre-test images. A 20.57-second follow-up received 1,963 input and 41 power
reports with zero rejects/overruns, RF off. This qualifies that historical
fault/reboot/recovery path. Direct maintenance handoff preserved GPIO and could
leave Linux running; a real watchdog reset did not provide that continuity.
The later [CRSF power-cycle experiment](../experiments/g11-crsf-electrical.md#controlled-module-power-cycle)
again observed MCU watchdog resets accompanied by Linux reboot.

## Earlier Linux-owned shutdown coordinator

The October 24 Linux inspection found startup scripts but no configured init
shutdown action or reverse service-stop sequence. Persistent writable UBIFS
mounts were `/`, `/config`, `/customer` and `/appconfigs`. UI exit or generic
BusyBox halt therefore did not establish a completed MCU power-cut handshake.

In the resident-loader design, short presses toggled manual screen blanking.
Touch woke a manually blanked screen and consumed that entire touch. A short
press followed by a two-second hold, with shutdown enabled and MCU capability
present, initiated saving. Only a clean settings save produced UI exit 42.
The supervisor then acquired UART/session ownership, synchronized and remounted
all four persistent filesystems read-only, checked the remounts and issued
PREPARE/COMMIT through its helper.

Coordinator exit 1 meant no commit was attempted and changed mounts had been
restored. Exit 3 meant commit might have occurred; exit 4 meant rollback failed.
The latter two refused an automatic writable restart. A reversible prepare/cancel
test passed on hardware with unchanged Linux boot ID. It did not perform a
physical power cut. The accompanying hidden soak record showed advancing
input-only samples, RF off and no new UART overruns; it was not an off/wake test.

## Autonomous MCU gesture and policy

The later MCU reads PC13 directly as an active-low pulled-up input, independently
of ADC/model ownership. Maintenance can therefore request shutdown without a
selected model. Each read validates GPIO clock/mode/pull; configuration loss
cancels the gesture and pending shutdown without stopping control.

After a debounced release, a short press/release followed by a two-second hold
within five seconds creates a nonzero physical request ID. A standalone hold
does not authorize shutdown. Continuing the normal gesture's hold to ten
seconds forces off regardless of Linux. A startup-held button must first be
released and pressed again. Debounce is 30 ms; a 100 ms sampling gap discards
partial gestures. MCU microsecond timing handles ordinary unsigned wrap.

| State | Control | Transition/expiry |
| --- | --- | --- |
| Idle | Continues | Physical gesture creates request |
| Requested | Continues | Linux PREPARE; expires after 15 s |
| Prepared | Continues | Linux saves and sends COMMIT; expires after 15 s |
| CommitWait | Continues | Matching successful reply must reach hardware UART TC within 2 s |
| Off | Stops | Final runtime save, then power-off/wake driver |

CANCEL or expiry returns to Idle without stopping RC. No Linux heartbeat
deadline governs autonomous RC. While shutdown is pending, unrelated model
STOP/activation and configuration-flash operations are refused. IDs increase
without wrapping within a boot.

A successful COMMIT response being queued is insufficient: only actual
transmit-complete on an idle, fault-free UART authorizes Off for the same
physical request. A UART fault clears delivery authorization before
reconfiguration, preventing idle TC after an abandoned reply from causing a
late cut. Exact protocol retries do not repeat the state change.

## Autonomous shutdown messages

Message types 28/29 are request/reply, using the custom COBS/CRC envelope and
boot nonce, management generation, session and contiguous sequence checks.
Legacy power messages 17–22 are not consumed by the autonomous board.

The request prefix is 14 bytes: version 1, operation, 64-bit boot nonce and
32-bit management generation. STATUS=0 has no further body; PREPARE=1,
COMMIT=2 and CANCEL=3 append a four-byte physical request ID. PREPARE acts only
on Requested; COMMIT only on Prepared. Remote commands cannot create a physical
request. STATUS also requires the negotiated management connection.

Every reply has 32 payload bytes, with multibyte integers little-endian:

| Offset | Field |
| --- | --- |
| 0–3 | Version, echoed operation, configuration result, shutdown state |
| 4–11 | MCU boot nonce |
| 12–15 | Management generation |
| 16–19 | Physical request ID; may retain last ID while Idle |
| 20–23 | Age in pending/off state, µs; zero while Idle |
| 24 | Flags: bit 0 valid button configuration, bit 1 debounced pressed, bit 2 forced off |
| 25–31 | Reserved zero |

These fields protect sequencing and identity; CRC and nonces are not
cryptographic access control.

## Linux save and terminal commit handling

The sole UART worker polls status every 200 ms; power status has a separate
500 ms freshness window even without an active model. After UI startup, a
Requested state opens confirmation. Its callback retains the exact boot nonce,
management generation and request ID. Confirm sends PREPARE; cancel sends
CANCEL. Startup, observation and UI destruction do not request a mutation.

After PREPARE, the UI exits its event loop and joins simulator, model, video and
Wi-Fi workers. Normal simulator closure saves editable settings. A nonzero
storage-dirty mask prevents COMMIT. The configured data filesystem and `/`
must both successfully open, `syncfs` and close. Save errors, interruption,
session changes or expiry refuse COMMIT. A post-close failure exits 1 for the
supervisor to handle; cancellation is best effort, with MCU expiry as fallback.

The worker re-observes the request before each mutation. After attempting
COMMIT, either successful or uncertain outcomes are terminal: only exact
same-session retries are allowed, with no following status query, automatic
reconnect or fresh-session replay. Ordinary quit/SIGTERM does not commit, and
this autonomous path never uses legacy exit code 42. Successful target `syncfs`
alone is not proof against every writeback or power-loss failure.

## Final save and off/wake mechanics

After COMMIT delivery or deliberate forced off, the MCU stops control and all
producers, captures live trims/GVars/persistent switches/timers and commits its
selection/runtime journal. This final-save gate is unavailable during active
control. Failed final writes do not block shutdown; journal integrity protects
the previous valid page. There are no periodic control-time flash writes.

Off masks interrupts, invalidates normal poll/IRQ ownership, stops ADC/DMA and
both UARTs, then invokes the power driver. Its traced sequence makes PA2/PA3
analog without pulls, lowers PA10, waits 200 ms, then lowers PB1/PB8 while
preserving unidentified outputs and SWD pins. The MCU feeds its watchdog while
waiting for a fresh debounced release and two-second wake hold. The later
autonomous path restores power and enters direct application startup with the
16 KiB RAM profile; the historical version re-entered its resident loader.

If the driver unexpectedly returns, normal work and watchdog feeding do not
resume. ARM and native tests cover policy, replay, expiry, UART delivery,
save errors and off/wake ordering with emulated boundaries. Full physical
rail behavior, complete-pair wake, forced-off storage outcomes and broader
power-loss durability remain open.
