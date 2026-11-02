# G11 stock power signals and reset behavior

[Firmware](README.md) · [G11](../devices/skydroid-g11.md)

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

## Linux shutdown boundary

The October 24 Linux inspection found startup scripts but no configured init
shutdown action or reverse service-stop sequence. Persistent writable UBIFS
mounts were `/`, `/config`, `/customer` and `/appconfigs`. Application exit or
generic BusyBox halt therefore did not establish a completed MCU power-cut
handshake. Physical rail timing and the complete stock off/wake sequence remain
unqualified by these records.
