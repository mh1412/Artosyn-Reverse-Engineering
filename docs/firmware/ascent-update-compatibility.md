# Ascent PC-tool dispatch and cross-model update boundaries

[Firmware](README.md) · [USB framing](../protocols/ascent-usb-management.md) ·
[Air-unit updater](ascent-air-unit-updater.md)

Offline inspection of Caddx PC Tool 2.0.40 and 2.2.9 confirmed that standard
Ascent VRX and VRX Pro use the same normal PC-side file-transfer state machine.
Their different firmware containers do not require different upload framing:
the PC can transport opaque file bytes while each device validates and programs
its own image. This October 7, 2026 result does not establish identical device
updaters, keys, partitions or commit behavior.

## Application identity and extraction

| Artifact | Bytes | SHA256 |
| --- | ---: | --- |
| 2.0.40 Windows application | 26,395,648 | `bc7beb450a38a8a5a921e9ff4b582126ab7e6c6f53ea77cfc8744c7dae26c605` |
| 2.2.9 distribution ZIP | 133,146,398 | `76dfed39449b3219e98114981acd0d3d2ae2c425b7445f8508a2699f62f3c3e6` |
| 2.2.9 setup executable | 133,651,472 | `8f1a7ce3c24873f6e8068eb10543021247000e580d0a9d688e38b720cc28b11a` |
| Recovered 2.2.9 application | 58,341,376 | `cb6cbfb2bb1606a7f018518c24adf6e459620568fe04f22f7e26619b0bd16dba` |

The Inno Setup 6.7.0 payload required raw LZMA2 decoding and reversal of CALL/JMP
preprocessing in 64 KiB buffers before extracting the embedded application ZIP
record. Declared size and CRC matched and CLR metadata was readable. An earlier
partial raw-ZIP carving failed and was excluded. Neither application nor installer
was executed for this research.

## Recognition, session ownership and transfer

Both versions explicitly recognize `ascent_vrx`, `ascent_vrx_pro` and `cx485_pro`.
These are code branches, not inferences from resource pictures. Discovery routes
Ascent devices to `AscentUpdataFrm` and `UpgradeProcessFSM`.

Version 2.0.40 creates `UsbSerialportFSM` with protocol type 2. Version 2.2.9
transfers the discovery session to the upgrade owner and requires
`LegacyFsmSessionTransport`, obtaining its serial FSM and constructing the same
upgrade FSM. An `ArTransportV2` type exists elsewhere, but that does not establish
a different normal upgrade path; `NewProtocolSendRebootClean` in the inspected
form is a return-only stub.

Both `UpgradeProcessFSM.Start` implementations construct a transfer session with
remote path `/tmp/pc/` plus the selected image basename and begin clean reboot.
The subsequent sequence remains information, clean reboot, session preparation,
metadata, data chunks, FileEnd and progress. Metadata is 328 bytes and includes
MD5, signed file length, save-as-file and two paths. Chunks come directly from
the selected file stream: this path is not an image repacker or signer.

The normal chunk default is 1 MiB, constrained by the receiver's reported maximum.
PC Tool 2.2.9 additionally has model-specific reconnect policy:

| Model names | Default delay |
| --- | ---: |
| Standard VRX | 9,500 ms |
| VRX Pro / `cx485_pro` | 12,000 ms |
| Max/HF/WF/Cine receivers | 12,000 ms |
| Lite / Lite+ | 8,000 ms |

These are software defaults, not measured boot durations. VID-based error/reboot
handling and negotiated transfer limits can also differ. A shared protocol does
not imply identical timing or target behavior.

The PC requests `/tmp/pc/` for Pro too. Actual placement/ACK behavior was tested
on standard Ascent; no Pro filesystem write was observed in this comparison.
The inspected CIL literal sweeps completed with zero unreadable methods and
found no Avatar model literal. A Walksnail UI string or unused goggles protocol
enum is not positive evidence for an original Avatar upgrade branch. This does
not exclude another tool, version or service route.

## Package and platform comparison

| Studied input | Format/platform evidence | Limit |
| --- | --- | --- |
| Standard Ascent 17.5.3 live / 17.5.8 reference | AR9311; ASW prefix plus modern OTRA; external RSA public-key validation; UBI/SquashFS and UBIFS | Same key verified within these standard-Ascent versions, not across product families |
| H Sky/Lite 17.5.8 | ARMv7/musl; ASW component envelope; traced user-space transfer MD5 and payload CRC; bank selection through `ar_bsel` | Does not determine every air-unit or BootROM trust chain |
| Older Avatar `Avatar_Gnd_23.23.4.img` | Pinned extraction sources describe OTRA v4, nested boot images, kernel initramfs and JFFS2/ext4 application data under `/local` | Not a verified specification for every AvatarSE VRX release |

The older Avatar schema places hash size at OTRA +4, signature size **in bits**
at +6 and version at +39. The studied modern Ascent schema instead puts header
version at +4, hash size at +10 and signature size **in bytes** at +12. A common
magic string cannot select the correct parser.

The reviewed Avatar extraction source was `fpv-wtf/ar-firmware-tools` at
`74a9c3573388660c2600c0d2ecc4045ce0698278`, identifying the historical research
input rather than its latest implementation.

The pinned Pro extractor identifies `Ascent_VRX_Pro_18_21_10.img`, with reported
SHA256 `8e06ee54db882ac8d8c84a96f94bbaf87e245ddb0cc8e59dab460e10cd898389`.
That hash was reported by the reviewed source, not independently recomputed from
a downloaded Pro image in this study. The extracted objects include `uboot`,
`boot` and UBI rootfs. Alternate-OS SD boot on that platform is separate from
the manufacturer's SD-file flashing procedure.

## Archived manual workflows

These are the model/version-specific instructions recorded in the research,
not update operations performed during the comparison:

| Manual | SD update trigger and approximate duration | PC workflow documented |
| --- | --- | --- |
| Avatar FPV VRX quick-start V1.0 | Matching `AvatarSE_Gnd` image, LINK eight seconds, about six minutes | Not described in that manual |
| Ascent VRX quick-start V1.0 | Matching `Ascent_G_Gnd` image, LINK eight seconds, about one minute | Caddx PC Tool with USB and external DC |
| VRX Pro user manual V1.1 | Matching model image, LINK fifteen seconds, about sixty seconds | Caddx PC Tool with USB-C and DC |

The common copy/button/program/reboot pattern does not establish the same physical
trigger timing, staging directory or writer. Absence of an Avatar USB procedure
in one manual is not proof that no service/recovery path exists. The standard
receiver's eight-second documented button action was not tested by initiating
an update in this comparison.

## What equivalence would require

Transport, staging, image parsing, authentication and programming are separate
boundaries. Cross-model equivalence requires the actual updater/startup handlers,
key bytes and a valid-package verification, followed by target partition/commit
evidence. Shared filenames, symbols or PC forms cannot supply those proofs.
The detailed [receiver](ascent-stock-updater.md) and
[air-unit](ascent-air-unit-updater.md) traces already demonstrate why each boundary
needs its own scope.
