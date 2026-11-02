# Ascent Lite script-upload incident and verified flash repair

[Experiments](README.md) · [Ascent Lite](../devices/ascent-lite-vtx.md) ·
[NAND representation](../firmware/sky482-nand-layout.md)

On October 16, 2026, script-upload experiments left the tested Sky482 unable to
start normal USB or its radio link. Direct NAND inspection established two
truncated startup helpers. A restricted repair restored them, passed complete
raw readback, and was followed by normal boot, authenticated root access and
visible stock video. This is the completed result; earlier recovery plans and
“pending” observations describe intermediate stages.

## Scope and incident sequence

The unit identified as `Ascent_lite`, board `FPV-Ascent-Sky-482-V1.3-1.1`,
running `Ascent_H_Sky_18_21_10`. Normal USB was `1d76:0101`, SerialGadget/CDC ACM.
The management endpoint answered CRC-valid OTRA device-information command
`0x3c`. It was not an established Linux shell.

The unmodified runtime command `0x3a` successfully switched serial to USB
Ethernet. The VTX responded at `192.168.3.104`, with a Telnet login on port 23;
port 22 refused connections. Root required an unknown password. The Ethernet
descriptor used `1d6b:0002`, also used by USB root hubs, so physical topology and
the transition from the identified VTX mattered more than that VID/PID alone.

Reference analysis suggested execution hooks through
`/usrdata/fpv/temp_measure.sh` and `config_ether.sh`. File metadata/data commands
`0x73`/`0x74` acknowledged marker-script uploads, but the expected temperature
marker did not appear. A stock helper was uploaded again without live readback.
A second wrapper intended to restore the USB script, create a RAM-backed root
password overlay and invoke normal Ethernet setup was then uploaded.

The next `0x3a` disconnected serial without bringing Ethernet back. Full power
cycles left a solid red LED, no normal USB and no radio link. The attempted
password overlay and script self-restoration were never verified. Continuing
to the second hook without verifying restoration of the first left a critical
startup dependency unresolved.

No full firmware image, raw NAND erase or bootloader write was issued during
those access experiments, and update-completion command `0x75` was not sent.
That did not make the uploads nonpersistent: the later flash evidence confirmed
application-file changes. The precise upload failure mechanism was not proved.

Data ACKs could report length zero, with a current-byte count accumulating across
files while total bytes described the current file. Some were followed by an
unsolicited CRC-valid command-zero `unkonwn cmd` error. Such frames must be kept
separate from the next request's matched response. ACK acceptance alone did not
prove persisted content, executable metadata or successful execution.

## Diagnosis from the preserved flash

Two complete page-plus-spare captures at different SPI rates matched. Reconstructing
the [controller layout](../firmware/sky482-nand-layout.md#reconstructed-controller-layout)
resolved initial filesystem extraction errors. Latest committed records and an
independent Linux UBIFS mount with journal replay agreed:

| File | Failed state | Preserved metadata |
| --- | --- | --- |
| `fpv/temp_measure.sh` | Zero bytes | root:root, mode 0775 |
| `fpv/config_ether.sh` | Zero bytes | root:root, mode 0775 |

Original compressed script contents survived in obsolete nodes. The primary
startup script still matched stock and repeatedly invoked the temperature helper
before USB and radio setup. Empty output therefore established a concrete startup
blocker. The numeric comparison against `1050` concerns the helper's raw value;
it is not an independently validated temperature in degrees. Repair restored
the original helper rather than bypassing that gate.

## Candidate construction and exact write boundary

The matching official 18.21.10 ASW package had SHA256
`78d9e73ce8ffe3b42991b12cdecca0b7b1140c3f8cbbba484deeedae51301907`.
Declared size and payload CRC passed. Its script bytes were reference inputs,
not pre-incident backups from the device.

The decoded primary application filesystem was loaded into a NAND simulator,
mounted with Linux UBIFS, repaired and remounted. Tree comparisons retained all
other file contents, ownership and modes. A separately authorized root-password
change replaced only root's SHA-512 hash in the primary root filesystem;
`/etc/passwd` already referenced shadow correctly and was preserved. The failed
temporary-overlay password was not the credential used for recovery.

The final candidate changed three file contents but **37 erase blocks**:
26 in rootfs0 and 11 in usrdata0. UBI/UBIFS journal recovery, allocation and
metadata account for the broader physical delta. Boot, environment, selector,
kernels, factory partition, alternate images and trailing space matched the
original capture before device boot.

The writer was a separate erase/program-capable XIAO build with a private,
compiled manifest of allowed block addresses and before/after CRCs. The normal
reader did not acquire a general write command. The host checked image SHA256
and every live block preimage before the first erase. For each allowed block,
the writer buffered the complete candidate, checked payload/current contents,
refused remapping/bad-block markers, verified erase, programmed pages in order,
verified returned bytes and restored configuration/protection.

Failures stop rather than trigger blind retries. Restoration failure latches a
fault. An interruption can still leave a partially updated filesystem; these
checks do not make NAND programming power-loss atomic. Eight recorded writer
simulation tests covered success, idempotence, wrong payload/preimage, unlisted
addresses, truncated upload, erase failure and program failure.

The preparation manifest retained `array_written=false` because it described
the candidate before programming. The subsequent hardware audit contains all
37 successful verified writes, and the final result records completion. These
records describe successive stages rather than contradictory device states.

## Physical and runtime verification

| Evidence | Result |
| --- | --- |
| Two original full captures | 138,412,032 bytes each, SHA256 `43a091f1b3f13dbdca62271700bff5feee47bb6163cb7045e25c2bc2e0f951ba` |
| Restricted write audit | All 37 block preimages checked; every write/readback and restoration passed |
| Independent post-repair capture | 65,536 pages; exact complete candidate match |
| Candidate/readback SHA256 | `666de2d7cf743494cde612999e1b1e1ba123197e4ba8acd5b0f87ef4a7427dc9` |
| Fresh flash probe | Configuration `0x18`, protection `0x7c` |

The first reconnection still showed solid red and no USB. After the owner
resoldered the flash connection, green blinking and normal USB returned. This
sequence does not identify the exact electrical fault during that first attempt.

A CRC-valid identity query again reported the expected board and 18.21.10.
The stock `0x3a` command, without further uploads, enabled Ethernet. Telnet
authenticated as root with the new credential. Live checks reported Linux
5.10.221, `fpvbsel=0`, `console=ttyAMA0,115200` and a running getty on that UART.
These live findings supersede the earlier unverified reference console name.

| Restored device file | Bytes | Owner/mode | SHA256 |
| --- | ---: | --- | --- |
| `/usrdata/fpv/temp_measure.sh` | 804 | root:root, 0775 | `e879b16ee39c18ceab62482964f7f4df0503f14ce0f9bda24fcc49986c15db0b` |
| `/usrdata/fpv/config_ether.sh` | 1,473 | root:root, 0775 | `132fdd667575ae499f5bee2abbc7869a8c50dfeafe79ba81279808e886bc5188` |

Root UBIFS mounted read-only; factory and primary application volumes mounted
read-write. The factory journal recovered normally. The live updater,
`daemon_sdiov12`, `ar_ldyhs_sky`, MP service and Telnet daemon were running.
Normal boot can update writable filesystem metadata, so the pre-boot whole-chip
preservation result is not a claim that later running flash remained identical.

Both stock radio queries subsequently returned state 2. An eight-second
1920×1080 HDMI-to-USB capture contained 401 MJPEG frames: the two-second sample
was black, while the six-second sample showed the camera scene, receiver UI and
a low-RF-power standby message. The record establishes visible stock video,
not uninterrupted video from startup, a controlled freshness challenge,
measured latency or long-run reliability.

## Recovery paths that remained unverified

Holding Bind/update while applying USB power after complete power removal
briefly exposed `3361:d001`, USBBurn. No BootROM payload was sent. Enumeration
proved a recovery entry point responded, not DDR initialization, a usable flash
loader or Linux boot.

A Windows bundle with CADDX PC Tool 2.2.9 and HiSilicon ToolPlatform CAM 5.7.14
was prepared but not run against this unit. A Hi3516CV610 profile and intact
archives did not provide board-specific DDR/loader assets or a demonstrated
readback procedure. BurnTool's Upload facility still required a working device
environment; its Burn Fastboot operation was documented to write flash after
RAM loading. It was not a read-only diagnostic step.

The ordinary ASW update file is not a complete BootROM kit or raw NAND image.
Neither the unused Windows path nor a third-party bootloader-rewrite report
explains the repair actually performed here. The verified result was targeted
filesystem repair with reconstructed controller ECC and full readback.
