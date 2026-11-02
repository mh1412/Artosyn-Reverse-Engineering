# G11 stock MCU update protocol

[Firmware](README.md) · [G11](../devices/skydroid-g11.md)

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
specific to the inspected image. GPIO C/F can retain inherited state
across a software jump, and an already enabled watchdog is not disabled by
branching into vendor code.
