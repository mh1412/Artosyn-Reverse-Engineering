# G11 MCU update and recovery history

[Firmware](README.md) · [G11](../devices/skydroid-g11.md) · [Current memory layout](g11-mcu-memory-and-models.md)

## Three different update paths

Stock updater analysis, the October 24 custom serial resident and the later
direct SWD image describe different firmware generations. The resident was
physically installed and tested, then removed for direct application boot at
`0x08000000` on October 25. Its serial uploader and relocated application
layout do not apply to the later autonomous segmented image.

This page preserves protocol and recovery findings, not a command sequence to
flash an unidentified current image. Later model/mirror/cold-code regions are
documented in the [current memory study](g11-mcu-memory-and-models.md).

## Stock Linux uploader

The inspected installed HAL SHA-256 was
`a8a68e5a23362c8ff22c7f89c02421292bbc6b6430b4ef7ff7f7230d22c263d9`.
It loads a raw application file, pauses its asynchronous receiver and uses
the existing `ttyS1` at 115200 8N1; no USB or baud switch was observed. No
signature check was found in this Linux path, which does not establish what
the then-unavailable MCU bootloader validated.

Vendor envelopes contain ASCII `fengyingdianzi:` (15 bytes), command, payload
length, payload and XOR of all preceding bytes. The captured state machine
at `0x14f04` follows this exchange:

| Phase | Host sends | Expected response |
| --- | --- | --- |
| Identify | A1, payload `52 01` | A0 application or A1 bootloader |
| Request boot from app | A5, ASCII `BOOT` | A5, ASCII `OK` |
| Confirm entry | A1, `52 01` | A1 |
| Raw handshake | `5f 00 00 f5` | `e5` |
| Description | `5e`, BE16 length, u8 block count, `e5`, XOR | `c5` |
| Data | 2,048-byte blocks padded `ff`, trailing XOR | `c5` continue, `55` resend, `aa` finished |

Block count is ceil(length/2048). The host can send another all-FF block if
the receiver requests continuation after declared data. This does not reveal
the bootloader's actual erase coverage. No arbitrary flash-readback command
was identified, and a downloaded application is not a backup of device data,
options or boot code.

Downloaded 0.5 and 0.8 both jump via the table at `0x08000000` after power/timer
actions and metadata programming. Their helper at `0x08003dce` erases page
`0x08001c00`, writes version 5 or 8 at +0, `0xabcd` at +4 and a runtime halfword
at +8. The version-dependent literal is part of the observed vendor sequence,
not a universal custom handoff contract. GPIO C/F can retain inherited state
across a software jump, and an already enabled watchdog is not disabled by
branching into vendor code.

An application-side compatibility prototype implemented only the three-frame
native entry exchange in a fixed five-second window. Captured ARM HAL
instructions and an ACK-only model checked 15 blocks plus forced retransmission.
That model did not implement physical flash. The compatibility parser was later
removed from production runtime in favor of the custom resident.

## Historical installed resident layout

The first custom resident was installed by Pico SWD while preserving vendor
boot and device data. The production allocation within the lower 64 KiB was:

| Inclusive address range | Purpose |
| --- | --- |
| `08000000–08001fff` | Vendor boot/metadata, 8 KiB |
| `08002000–08003fff` | Custom resident, 8 KiB |
| `08004000–0800d7ff` | Application capacity, 38 KiB |
| `0800d800–0800dbff` | Preserved device data |
| `0800dc00–0800dfff` | Application validity manifest |
| `0800e000–0800f7ff` | Preserved device data |
| `0800f800–0800ffff` | Two boot-nonce pages |

An earlier 44 KiB application fixture overlapped installed data and was replaced
before production use. Ordinary updates changed only image-covered application
pages and the manifest, without option-byte programming or resident self-update.
These were software range restrictions, not hardware isolation from faulty code.

The resident held RF off, serviced its watchdog and waited for explicit Linux
upload or RUN; a valid stored image did not auto-run. A real watchdog reset
also rebooted Linux, whose persistent Wi-Fi/UI services restored access.
Later direct maintenance handoff preserved GPIO and Linux power. ACK delivery
alone did not prove a loader or application had started.

## Image transaction and manifest

The update engine first validates size and layout, erases/verifies the entire
manifest, then erases/verifies only pages covering the declared image. Sequential
chunks of at most 128 bytes are programmed with halfword readback. A lost-ACK
repeat is accepted only if stored bytes match; an odd length is permitted only
for the final chunk, padded with an erased byte.

FINISH reads the whole image and validates reflected CRC32/IEEE plus a 65-entry
vector table: aligned SRAM MSP, Thumb reset and nonzero handlers within the
declared image after its vector table. CRC is integrity, not authentication.
Boot independently repeats manifest/CRC/vector checks. The 32-byte LE manifest is:

| Offset | Field |
| --- | --- |
| 0 | u32 `0x55313147`, ASCII `G11U` |
| 4 / 6 | u16 format 1 / record size 32 |
| 8 / 12 / 16 | u32 application address / byte length / image CRC32 |
| 20 | CRC32 of bytes 0–19 |
| 24 | Reserved `0xffffffff` |
| 28 / 30 | u16 commit `0x3cc3` / complement `0xc33c`; offset 28 written last |

CRC initial/final XOR are `0xffffffff`, with check value `0xcbf43926` for
`123456789`. An I/O fault latches the transaction off; replacement starts again
with manifest invalidation. The design does not retain the old application
through an interrupted replacement. It preserves an independent resident able
to accept another complete upload. This historical manifest must not be
confused with the later [aligned model commit fix](g11-mcu-memory-and-models.md).

## Resident protocol version 2

The custom resident uses exclusive 115200 8N1, COBS plus zero delimiter and
CRC16-CCITT (initial `0xffff`, polynomial `0x1021`). Version 2 distinguishes it
from control version 1. Requests have version/op at 0/1, LE32 token/sequence
at 2/6, up to 64 payload bytes at 10 and trailing LE16 CRC; maximum wire length
is 78 bytes.

Replies set operation bit 7 and echo token/sequence. Raw reply length is 46:
status at 10, boot nonce u64 at 11, generation u32 at 19, app start/end/manifest
u32 at 23/27/31, image state at 35, received/declared lengths u32 at 36/40,
CRC16 at 44. Status 0–6 means OK, bad request, busy, session mismatch, sequence
mismatch, flash failure, invalid image respectively.

| Op | Name | Request payload |
| --- | --- | --- |
| 1 | HELLO | Empty; nonzero token, sequence 0 |
| 2 | OPEN | 12-byte challenge from HELLO; sequence 1 |
| 3 | BEGIN | Challenge + LE32 size + LE32 CRC32 |
| 4 | DATA | Challenge + LE32 offset + 1–48 data bytes |
| 5 / 6 / 7 | FINISH / STATUS / RUN | Challenge only |

Challenge is persistent boot nonce plus per-offer generation. A same-token
HELLO retry reuses its offer; other tokens cannot take over until five-second
inactivity expiry. A new offer after expiry has a new generation, and reset
requires a new nonce. Generation/sequence wrap is rejected. Commands advance
contiguously; only the exact immediately preceding request replays its cached
response, including errors. Reusing a sequence with changed bytes is rejected.
Partial receive input expires after 100 ms; stale sessions cannot mutate flash.

Host stop/wait sequence is BEGIN, STATUS until receiving, DATA, FINISH, STATUS
until complete and optional RUN. An I/O fault stays latched until loader reset.
RUN requires valid image and drained ACK; a lost ACK leaves its outcome unknown,
because the application may already own the UART. RUN-existing uses only
HELLO/OPEN/RUN and performs no application programming. Inspection's HELLO is
not fully passive: it reserves a five-second discovery lease.

## Physical erase-pacing correction

The first real control upload faulted during manifest erase before application
data was written. SWD watchpoint evidence showed loader error 3, UART fault 1,
flash fault 0, image state ERASING. Flash stalls had delayed instruction fetch
and UART service.

The corrected service makes STATUS execute exactly one queued flash step
**before replying**. Foreground polling handles session expiry, not autonomous
erase progression after replies. BEGIN/FINISH acknowledge state changes, DATA
completes/readbacks programming before responding, and cached duplicate STATUS
does not repeat its step. A compliant stop/wait host therefore does not transmit
its next request during erase. Default 750 ms request timeout accommodates the
recorded datasheet's up-to-300 ms page erase; early retries or unrelated writers
can still fault the link.

The adapter processes bounded RX and requires reply TC within 200 ms. UART/FMC/
nonce failure stops watchdog feeding. App entry validates vectors, sets VTOR/MSP,
selects privileged Thread/MSP and branches with a trap return address. An
unexpected return is fatal. Power GPIO and the running watchdog are preserved.

## Physical recovery evidence

A committed 264-byte non-starting application was acknowledged and entered;
subsequent observations found the resident running again after watchdog
recovery. A separate 48-byte partial upload followed by reset had an erased
manifest and erased image-covered remainder; vendor boot, resident and reserved
device data were unchanged. RUN-existing rejected operation 7 with status 6.
A complete 12,976-byte replacement then restored fresh input-only controls.
These were not supply-loss tests during an actual programming pulse.

Repeated uploads and direct maintenance passed without Linux reboot, while
fatal/watchdog resets did reboot Linux. Clean builds under the recorded ARM
GCC 13.2.1 environment matched previous SWD reads:

| Artifact | SHA-256 |
| --- | --- |
| 16,040-byte control app | `8f12abef387fbd9220168403468a59bcef6d517f783fb63a952ecd5ca6638670` |
| 8,130-byte resident file | `3b4b45a71ae1edf7b0f845ccfdcb21680c90465458e8b936292afb98bf3316cd` |
| 8,132-byte resident read with alignment padding | `ac0ab2ed9c94a3239f171783a0587767ce0546cca4944998088cab7bbd0bb083` |

This is recorded-environment reproducibility, not a cross-compiler guarantee.
Native uploader smoke tests without UART access and incomplete linker fixtures
are weaker evidence than these later physical operations.

## Transition to direct SWD boot

The October 25 direct image replaced vendor/resident code in the first 16 KiB
and booted at `0x08000000`; serial-update capability was disabled. Initial
16,052-byte image SHA-256 was
`1da64e6a26b3e2b1eb47d7add71076b26cb75e8803768cb184598241fdea355c`.
The dated operation used Pico CMSIS-DAP, OpenOCD's stm32f1x driver, an explicit
64 KiB bank, no RAM work area and watchdog freeze while halted. Readback
verified bytes beyond the written first 16 KiB unchanged before reset and
matched the running image afterward. Probe capacity guesses were overridden;
they were not physical capacity qualification.

A direct-start marker bypassed the old uploader/runner. Linux rebooted during
installation, and Wi-Fi/UI returned. Later updates fixed
[long-session timestamp admission](g11-uart-and-session-lifecycle.md#long-session-timestamp-defect)
and a [completed-DMA timeout defect](../soc/g11-gd32-acquisition.md#raw-inputs-and-dma-publication); an attempted 75-minute
soak had stopped after 124 seconds, so it was not endurance evidence. Later
autonomous builds used physically qualified 16 KiB RAM and upper flash,
with segmented fast/cold code and model storage. The original 64 KiB bank and
first-16-KiB operation cannot be reused as a description of those later updates.
