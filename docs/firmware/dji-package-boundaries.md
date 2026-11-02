# DJI package extraction and encrypted boundaries

[Firmware index](README.md) · [Receiver architecture](../systems/dji-receiver-comparison.md)

## Research scope

This records the **October 12, 2026** extraction study. Versions below identify
the studied artifacts, not the latest available releases. The result is an
extracted Goggles 3 application filesystem and inspected O4 update metadata;
the O4 air-side executables and root filesystem were **not decrypted**.

## Goggles 3 application software

The studied `zv902` v01.00.1400 archive is 281,845,760 bytes:

```text
SHA-256 d52f3ed1f95bf6a5cf54629bb0c95198774c581a42ae72ddab6f3fbba60a3e9e
```

Its hash matched the third-party download archive's metadata. The tar contains a
configuration manifest and modules `1403` v48.00.00.84, `2803` v22.11.11.52 and
`2805` v10.00.62.24. Module 2805 contains an Android 9 `e3t_zv902` OTA. A later
v01.00.1500 entry was observed during the study, but its download returned HTTP
404; that version was not analyzed.

| Reconstructed image | Bytes | SHA-256 |
| --- | ---: | --- |
| System | 654,311,424 | `4940622541edd2e58ce1e6e3df8d2c31ef75e20372d81c33fd0b37acd044b295` |
| Vendor | 150,994,944 | `993920e29c279c8e58233fb19da6240f20dca00512812b6975ec5d69945b1571` |

The recorded verification covered OTA SHA-1 checks for new-data and zero-filled
ranges, exported entry types and regular-file sizes. System exported 1,761 regular
files and 167 symlinks; vendor exported 238 regular files. Original ownership was
recorded in the manifest because an unprivileged export could not apply it.
The vendor image retained its original inode-bitmap padding warning without repair.

The outer IMaH payload was plaintext. Forced extraction bypassed an unsuccessful
signature check using the available key; payload checksums still matched. This
does **not** establish DJI signature authenticity. The separate `normal.img`
boot kernel/LRFS ramdisk remains TBIE-encrypted: the tested public keys did not
produce a valid kernel header. System/vendor exports are not a complete boot root.

## O4 standard and Pro packages

| Property | O4 Air Unit | O4 Air Unit Pro |
| --- | --- | --- |
| Device code | `za530` | `za5305` |
| Formal archive version | 01.00.0400 | 01.00.0700 |
| Recorded release date | 2026-06-25 | 2026-07-02 |
| Archive bytes | 166,737,920 | 167,413,760 |
| Module version / internal date | 10.00.17.07 / 2026-04-14 | 10.00.17.06 / 2026-06-23 |
| Module manifest name | `ZA530_E3T` | `ZA5305_E3T` |
| Main signed member bytes | 166,730,816 | 167,404,352 |
| Payload bytes | 166,730,187 | 167,403,717 |
| Antirollback metadata | 1 | 2 |

Archive SHA-256 values:

```text
standard 6cc443dc683743ccc0dee9c5cbf6412ab76b5bfae2b95f5a79393897056bfa6d
Pro      b7e53805b5a29b8d84a50de51c247bf59ed801de5923bda5a7b006fffc3b9219
```

Both archives contain exactly a configuration manifest and one signed `0802`
module. One update module does not imply one physical processor. `E3T` labels do
not establish the air-side OS, instruction set or services. Formal release date,
manifest schedule and module build date are separate fields; standard's manifest
`from` date is May 7, not its June 25 release date. Pro release notes format its
version as `01.00.07.00`, while the archive uses `01.00.0700`.

The main container has the following recorded fields:

| Field | Value |
| --- | --- |
| Magic / header version | `IM*H` / 2 |
| Header / signature bytes | 224 / 384 |
| Name and chunk ID | `0802` |
| Type | `E3T` |
| Authentication / encryption key labels | `PRAK` / `UFIE` |
| Authentication algorithm field | `0x00040003`, not fully interpreted |
| Chunk count / flags | 1 / 0 |

The manifest is `CNFG`, chunk `0000`, with an empty encryption-key field; its XML
is readable. The study checked archive hashes, tar/member sizes, declared lengths,
ciphertext/payload hashes, encrypted-data checksums, plaintext manifest checksum
and XML, and main-module MD5/size against the manifest. Historical PRAK keys and
an additional recovered public RSA candidate did **not** verify the signatures.
Internal consistency and signature authenticity are separate conclusions.

## Failed decryption is a retained result

At `dji-firmware-tools` revision
`195692263c2684cf1ddc4995f2736be6c0fb135e`, six UFIE variants were tested against
the full encrypted payload using the tool's IMaH-v2 AES-CTR implementation:
2021-08, 2021-06, 2020-04, 2019-11, 2018-07 and 2018-01.

All failed the expected plaintext checksum for standard v0400, Pro v0700 and an
additional standard v0100 baseline: **18 failed full-payload tests**. The baseline
archive hash is `e3475c014967a835f6bb263dfdc7267a0d5e7b735b2764575ae4ee6c87e405a6`,
with module 10.00.11.75 dated 2024-12-19. Wrong-key output was quarantined and not
used as a root filesystem, executable or string evidence source.

An unavailable applicable key or unsupported encryption generation could explain
the failure. The results do not identify a particular missing key. UFIE update
encryption, TBIE encrypted boot/modem resources and RF-session encryption are
different boundaries; none establishes the others' cipher or packet format.

## What the plaintext manifest does establish

Module `0802` is marked `is_upgrade_center: true`, using operation library
`libeagle_md_up.so`, communication method `V1` and parameter `0x0801`.
That parameter is not a USB endpoint. `loader_needed`, `post_check_ver`,
`resp_get_ver`, `reboot_after_fail` and `reboot_notify` are true;
`support_multi_hw` is false. These are updater declarations, not executed tests
of recovery, rollback enforcement or board compatibility.

The receiver's O3/O4 modem profiles can be compared as stored bytes without
claiming to have decrypted them:

| File | O3 bytes | Standard bytes | Pro bytes | Standard and Pro byte-identical? |
| --- | ---: | ---: | ---: | --- |
| `ap.img` | 309600 | 317344 | 317344 | No |
| `cp.img` | 430368 | 509312 | 509312 | No |
| `dsp.img` | 359648 | 255648 | 255648 | No |
| `l1c.img` | 597856 | 644736 | 644736 | No |
| `scp.img` | 46816 | 46816 | 46816 | No |
| `cp_nvram.bin` | 1120 | 1120 | 1120 | Yes; also matches O3 |
| `rf_nvram.bin` | 10336 | 10336 | 10336 | Yes; also matches O3 |
| `share_info.bin` | 80 | 80 | 80 | Yes; also matches O3 |
| `ipc.bin` | 864 | 608 | 608 | Yes; differs from O3 |
| `pwr.bin` | 123248 | 123248 | 123248 | Yes; differs from O3 |

Different encrypted image hashes do not prove which instructions changed. Shared
default NVRAM files do not prove identical per-device RF calibration. These are
Goggles 3 receiver resources, not decrypted O4 air-unit partitions.

The comparison complements [Ascent update authentication](ascent-stock-updater.md)
and [cross-model update boundaries](ascent-update-compatibility.md): container
structure, checksums, authentication, decryption and hardware compatibility each
need their own evidence.
