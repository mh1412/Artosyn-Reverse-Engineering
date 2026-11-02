# AR8030 physical SDIO and RF bring-up

[Experiments](README.md) · [AR8030](../soc/ar8030.md) ·
[Loading and driver lifecycle](../firmware/ar8030-loading-and-driver.md)

The early Gnd485 18.21.10 experiments established execution on the physical
AR8030, reliable diagnostic SDIO exchanges and bounded RF peripheral/calibration
operations. Trials through 59 did **not** receive valid native RF packets. Later
[receive acquisition](ar8030-receive-acquisition.md),
[first video](ar8030-first-video.md) and
[Open8030 validation](open8030-validation.md) are separate milestones that
supersede the early notes' blanket “video not working” statements.

## Evidence and execution boundary

USB SSH reached the receiver's AArch64 Linux host. The replacement RV32 runtime
executed on its separate radio at `0x00204000`, using temporary RAM loads,
the stock bootstrap/configuration and stock SDIO loader. Direct radio UART/JTAG
was unavailable. Installed firmware/configuration and flash were not changed.

| Input | SHA256 |
| --- | --- |
| Stock 431,104-byte radio container | `dc58247910af564376350404383c5c411cbec21663873fba24e70c61a3584e3c` |
| Stock 412,736-byte main runtime | `3e454131a334ec422c0b2e8ae4924d6bda68cb61b70fd754da26412578b39c87` |
| Trials 20/21, 4,368-byte padded replacement runtime | `cce4a407522b49d4a8f34fb529523908f33546a4f6d8ecdd494e8394aadadcca` |
| Trials 20/21, 22,528-byte container | `781316096cf534d4221bd66f2ff7544e47a67fbd5572268e061a966908fb238e` |
| Trial 23, 4,848-byte padded replacement runtime | `9efd0798b46636bcaaa2be3910b3ad7491bce90dc6fb93d7d4d750089907320e` |
| Trial 23, 23,040-byte container | `00163da159221dc626afd0f1c37e763bf3c3ca4b37230225a6ae44ac7f2d5495` |

The initial native build selected RV32IMC/Zicsr/Zifencei and ILP32 soft-float,
with ELF RVC flags 1. That choice does not identify the original vendor ABI.
An AArch64 host codec test, QEMU protocol test, partial ISA emulator fixture and
physical radio challenge test exercised different boundaries. None substitutes
for an RF/video test.

## SDIO: a notification race hidden by instrumentation

The radio enumerated as `4152:8030` in ROM and `4152:8031` after loading. Early
runtime identity alone was misleading: several builds reached the running ID
but failed the first host write with `-5`. A custom `8032` identity prevented
the stock driver from binding. A stale payload checksum prevented another
candidate from leaving ROM; the first attempt failed before loading because
the host could not find `rmmod`.

The decisive ordering was:

1. The Linux probe claims the IRQ and writes function-1 interrupt enable `0x14=1`.
2. It then writes the event/mask register `0x68=0xf0`.
3. A premature native RX-room advertisement between those writes is cleared by
   the second operation. The MCU can show nine available blocks while the host
   sees no room notification.

The correction records MCU event `0x800` and waits for the high nibble of MCU
register `0x168` to become `0xf0` before publishing receive room. A simulated
regression separated IRQ enable from the later mask write. Trials 20 and 21
then passed with the short original-style CIS after independent radio resets.

| Trials | Result and interpretation |
| --- | --- |
| 1–12 | Loader, identity, initialization and grant experiments exposed separate failures; memory attributes and byte grants alone did not fix host writes |
| 13, 16 | Adding a longer `LIVE` CIS tuple allowed 4 KiB exchanges; changed timing masked the race |
| 14–15 | Removing the tuple and adding fences did not reproduce the success |
| 17–19 | Pending reads, late enable and a poll pause did not establish the fix |
| 20–21 | Explicit host-enable/mask handshake passed hello, diagnostics and eight fresh 4 KiB echoes per load |
| 22–23 | Four queued 4 KiB replies passed in addition to the independent heartbeat checks |

The trials 20/21 runtime and container bytes are identical. Each trial required
`ZEPH` hello, the diagnostic identity
`ar8030-zephyr/bringup;rf=unsupported;protocol=1`, and byte-for-byte comparison
of eight fresh Linux `getrandom` payloads with distinct message IDs. Both probes
exited 0. The measured 52–53 ms round trips include the host driver and probe;
they measure neither RF latency nor sustainable throughput.

Packaging manifests retained `boot_acceptance_verified: false` because packaging
preceded the hardware tests. The successful challenge receipts establish the
later acceptance. Conversely, later checkpoint source again contains optional
`LIVE`/diagnostic CIS instrumentation; the historical short-CIS result does not
mean every subsequent build has that exact CIS.

## Queued transfer ownership and instruction checks

The queued implementation has four TX entries, including the active transfer,
with independent 512-byte-aligned, 4,608-byte buffers. It copies a reply into its
owned buffer immediately and zeroes transfer padding. The buffer remains owned
until completion or abort. It acknowledges the observed completion before
publishing the next entry. A full queue increments backpressure and withholds
new RX grants; receiving a valid request while TX is active is otherwise allowed.

The diagnostic burst request `0x0c5a0001` asks for four payload echoes with
consecutive IDs. Passing that burst establishes this bounded ownership case,
not video transport. The reviewed source still uses single-core polling, with
CPU interrupts disabled; its TX counter advances on submission, not completion.
A four-entry queue does not establish general reset/error recovery, cache
coherency or a complete interrupt-driven HAL. The early 29,152-byte RAM build
and queued 47,616-byte build also differ from later RF/video builds.

Trial 23 tested exact instruction word `0xad8787f7` on 1,024 fresh inputs. It
swaps adjacent bytes within each halfword:

```text
((x & 0x00ff00ff) << 8) | ((x >> 8) & 0x00ff00ff)
0x12345678 → 0x34127856
```

That hardware result applies to the tested encoding. Partial models also gained
conditional moves and indexed loads/stores, with synthetic checks for signed
scaled increments and writeback order. Reserved/zero-base aliases, opaque cache
operations and a draft packed add model were not thereby validated as a complete
CPU architecture. Later instruction checks have their own evidence.

## Calibration passed before reception worked

Trial 24 first passed OTP/profile discovery, board mux setup, clocks, ordered
register defaults, 5 GHz calibration and PSRAM testing. Trials 36–46 repeated
these stages: chip profile 1, 704 ordered writes, calibration convergence in
three rounds with state 6, and PSRAM result `0xaa991007`. Receive monitoring used
CLIC pending-bit polling and four 64 KiB buffers. Zero stage status meant that
bounded operation completed; unsupported vendor RF commands did not become
successful merely because calibration worked.

The measured baseband page is `0x0a110000`. An earlier UART assumption at this
address was removed from the physical board description. The actual physical
console location remained unresolved; QEMU's UART is a separate platform fact.

The defaults export traced `0x2237dc` to before `0x21db3c` in a selected
DEV/mode-0/chip-1 clock context with initially zero emulated MMIO. Its 704
`<HHI>` records contain offset, reserved zero and value: 5,632 bytes, SHA256
`4ef051a55bd0ab8da661d32dd6fcaa565d580aacaf0155cab6504365db4f78fc`.
This preserves ordered stores, not a complete timed hardware initialization.
Live pointers, status and calibration observations must not be replayed as
static defaults. Stale processor-model exports and incomplete restore-tail
analysis were also superseded by corrected analysis runs.

## Failed receive hypotheses, trials 36–46

| Trial | Correction or control | Result |
| --- | --- | --- |
| 36 | Compensation field `0x48c00008` corrected to `0x048c0008` | Seven operations preserved state; no packets |
| 37–38 | Static user context, stock DEV sequence, clock/mode and PHY snapshots | No valid headers or packets |
| 39 | One-second monitors queried after 350 ms | Next start correctly rejected because the previous monitor was active |
| 40–42 | Wait 1.1 seconds, then compare slot-0 and broadcast receive | Invalid header/error activity at 5.839 GHz; no packets |
| 43 | Correct nonuniform PLL shadow placement | Activity moved to the configured 5.770 GHz; headers remained invalid |
| 44 | Fix both direction banks and static PHY/power fields | Same invalid-header result on the intended channel |
| 45 | Minimal prepare/receive/end and sync/data sequences | Invalid slot-0 headers; sync remained waiting |
| 46 | Match masks at `0xbac` and `0xbd4` to stock | Both read zero; still no valid headers or packets |

The compensation typo addressed `BB+0x48c0` instead of `BB+0x48c`.
Simulation was corrected to reject unmapped writes. Earlier cleared state did
not independently prove a CPU reset.

The PLL field `0x09381028` begins at byte `0x93a`: its encoded shift matters,
and adjacent slots do not form a uniform five-byte array. Correcting this map
invalidated the earlier interpretation of activity at 5.839 GHz as properly
tuned reception. At the intended 5.770 GHz, trial 44 still recorded 165
interrupts, header `0xc0000000`, raw error 7 and zero valid headers, RX events,
DMA completions or packets. Interrupt activity does not identify the VTX as
its source. A working stock observation had a valid header and nonzero raw
error `0xb`; error values alone are not a verified pass/fail decoder.

## Stock observers narrowed the search, trials 47–58

A project diagnostic `0x0c5a0018` accepts LE16 offset/length, at most 256 bytes,
within `0x0a110000..0x0a110dff`. It requires alignment and rejects reads during
active monitoring. Full 3,584-byte snapshots followed completed monitors;
this was not an arbitrary-memory API or a vendor command.

Further native trials added user buffer controls, closed-loop power settings,
second-bank identity fields, compensation and `0x0a1060c8 |= 3`. Physical-user
selection, bandwidth codes 0–5, sync/search termination and identifiers 0,
`0x66`, `0xcb` all failed to produce valid packets. Some fields persisted between
operations, so ordered operations were not independent cold starts. Trial 52
failed before upload; it is not a negative RF experiment. Trial 58 completed six
monitors and exited 25 to report failed reception.

Trials 53 and 56 instead ran **instrumented stock firmware**. Each changed only
the instruction at `0x246146` in bounded GET_REG: original `0x0a1106b7`, clock
observer `0x0a1046b7`, GPIO observer `0x0a10a6b7`. These redirected the read page,
retaining the bounds/read loop. The observer payloads were independently checked
against the original runtime; the changes do not constitute native RF success.

The observer payload hashes are
`42e38c3fa8fa44f1f9c7e295d0e96973b84faaa9ba988e3ed9005ce2ed60ac6e`
(clock) and
`7a991d43ba0d09e52b237e9aa9c247c810b855b2d67cff2114f11f5ed49f23d6`
(GPIO). Selected clock readbacks relative to `0x0a104000` were:

| Offset | Instrumented stock | Native |
| --- | --- | --- |
| `0x00` | `0x00001000` | `0x00001000` |
| `0x04` | `0x12001101` | `0x12001101` |
| `0x08` | `0x11011101` | `0x11011000` |
| `0x0c` | `0x10001000` | `0x10001000` |
| `0x10` | `0x80031001` | `0x80001000` |
| `0x14` | `0x11001100` | `0x11001100` |

Differences included peripheral clocks; native retained an OTP clock that stock
disabled. At GPIO base `0x0a10a000`, A/B pad words `0x04/08/0c/10` matched as
`0x21212123`, `0x23`, `0x21000000`, `0x20`; mux words `0x2c/30` matched as
`0x2334`, `0x11777`, and word `0x44` matched as `0x1f`. Bank-control word 0
differed, stock 2/native 0, in C-bank control. These comparisons narrowed the
search without proving all analog clocks or physical RF routing.

The matching Linux backend's SDK bandwidth enum is 0=1.25, 1=2.5, 2=5, 3=10,
4=20 and 5=40 MHz. Thus code 4 means **20 MHz**, correcting the early 40 MHz
interpretation. Its TX/RX values are 0/1 and AP/DEV values 0/1. Linux DWARF types
are not declarations recovered from the RISC-V firmware.

## Static receive-start correction, trial 59

REA 6.4.0 with Ghidra 12.1.4 explicitly rejected the custom processor profile.
The subsequent analysis deliberately used generic `RISCV:LE:32:default`/GCC;
it neither executed the input nor silently supplied complete custom semantics.
A bounded graph covered three procedures, 86 nodes and 100 edges, not the whole
startup graph. Incorrect nonreturn annotations on a division helper had also
hidden the continuation of startup; corrected custom-profile analysis supplied
separate corroboration.

The reconstructed path enables IRQ 5, performs PHY channel setup, posts LINK
message `0x200`, clears CF4 bit 21, prepares the frame/tuner and starts receive.
Its selected RX identity becomes zero when the relevant flag is clear. Trial 59
implemented that sequence but obtained only a command interrupt, with no valid
header or DMA packet, and exited 25. It still missed the shared DF0 identity
update corrected in trial 60. The later
[startup study](../soc/ar8030-receive-startup.md) distinguishes search message
`0x205` from maintenance message `0x206`; early shorthand conflating them should
not be carried forward.

## Restoration and stock-video measurements

Radio identity and a restored image hash were insufficient recovery criteria.
After trial 21, the same Linux boot remained running and the stock radio
returned as `8031`, but link-info queries returned `-1`. Another radio-only
restore delivered zero video bytes over six seconds. Full receiver reboot then
restored a passive six-second observation of 2,172 messages, 15,211,164 bytes
and 360 advancing frame IDs, with 1920×1080/60 metadata. That capture started
mid-GOP and lacked VPS; it was not an independent HEVC decode result. Separate
startup-capture attempts obtained no packets.

Trial 36 exposed another recovery defect: a forked CIS observer retained an
SDIO descriptor, preventing module removal and leaving ROM identity. Terminating
and reaping the observer corrected ownership; power cycling recovered that
incident. Later restore receipts and full-reboot checks remained separate from
native receive acceptance. After trial 58, reboot restored all 24 getters and
a 301-frame nonblack capture without cycling the transmitter.

Direct stock HDMI capture used a MACROSILICON `345f:2130` USB grabber at its
existing 1920×1080 YUYV/30 configuration. Initial QBUF failures and USB error
`-71` were resolved by resetting the grabber; that was not radio recovery.
A 30-second baseline delivered 901 frames at 30.002 fps, with median/p95/max
intervals 33.327/33.353/34.918 ms and no measured loss, short frames or timestamp
gaps. Four warmup errors were recorded separately. Concurrent radio sampling
made 240 getter calls without errors; their RTT is host/RPC timing, not RF delay.

An earlier baseline calculation included four zero-timestamp/error startup
buffers and was explicitly superseded. Its rate is invalid; the corrected
warmup-aware capture supplies the baseline above.

The content-aware follow-up also delivered 901 frames, but 67 measured frames
were black and the first nonblack frame appeared 3.252 seconds after capture
start, including warmup. Sampled luma fingerprints establish sampled variation,
not whole-image freshness or camera-to-screen latency. An earlier capture with
only two fingerprints and an “RF CONNECTED” HUD illustrates why transport cadence
and link labels alone cannot establish fresh camera video.

The final stock capture after the trial-23 socket-opcode probe is a separate
record: 901 measurement frames at 30.004 fps, 835 distinct sampled fingerprints,
67 near-black measurement frames and first measured nonblack arrival at
3.261 seconds. It recorded zero sequence losses, driver errors, short frames,
invalid timestamps or large gaps. Those are capture-output checks: raw YUYV
has no compressed decoder, so its driver-error count is not an HEVC decode-error
count. The accompanying six getter samples contained 24 successful queries;
both reports explicitly left replacement-radio parity unproven.

The failed startup-header captures also motivated a narrower loopback trigger:
an optional currently observed video TCP destination port, with source port
50000, can recognize payload even when it begins inside a baseband frame.
ACK-only traffic, other destinations and truncated payloads do not qualify.
The destination port is ephemeral and must be rediscovered after reconnecting.
The synthetic trigger regression passed; the two retained hardware attempts
still produced empty captures. Neither the trigger change nor the final stock
HDMI capture establishes independent HEVC decoding of that 18.21.10 capture.

The baseline comparator required matching grabber identity, geometry, format and
rate. Its 98% reference tolerance was a test policy, not a product specification;
it explicitly left full radio-firmware parity unproven. Full parity additionally
requires real lifecycle/session behavior, fresh video/control/telemetry,
reconnects and sustained measurements on the named board and firmware.

This consolidation independently checked retained container hashes/checksums,
observer edits and packed-default structure. Hardware, original emulator tests,
raw capture timing and visual inspections were not rerun for the wiki review.
