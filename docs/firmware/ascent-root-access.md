# Ascent stock root access and account overlays

[Firmware](README.md) · [Gnd485](../devices/ascent-vrx.md)

Root SSH was obtained on the investigated Gnd485 by changing writable startup
behavior and creating RAM account overlays. The original password was not
recovered, and the separately prepared rootfs patch was never installed. These
are distinct access methods and evidence histories.

## Initial authentication and reference-image work

The initial receiver ran 17.5.3 and exposed Dropbear 2019.78 over USB Ethernet at
`192.168.3.102`. SSH negotiation needed the `ssh-rsa` host-key algorithm option.
A credential from a different BetaFPV reference was rejected. Public Ascent
16.5.7 and official 17.5.8 contained the same MD5-crypt root hash; the original
underlying account file of the live 17.5.3 unit was not independently read before
its overlay was installed.

Bounded offline dictionary, product-pattern, decimal-mask and binary-string
searches found no match. The seven-lowercase-character job examined only 15.47%
of its space before the time limit; optimized mutation runs also had length
exclusions. Those failures do not prove an arbitrary password unrecoverable.
No additional online password guessing or external recovery submission was
performed during that phase. The later working credential was deliberately
selected for the lab installation, not discovered in vendor firmware.

Reference provenance mattered: changing only a download URL's filename returned
the same object, not a verified 17.5.3 image. The official 17.5.8 package was
197,377,611 bytes, SHA-256
`4c80b9af79a0075164a74cb6f673d5f0b9ad25dc736717f6b62bc493e362f1d6`;
its receiver image was 49,207,658 bytes. Offline inspection of the bundled macOS
PC tool found no useful plaintext SSH credential among 287,319 candidate strings.

## Uninstalled SquashFS candidate

An offline 17.5.8 rootfs edit replaced one same-length password-hash literal in
LZ4 data and then compared the complete filesystem traversal:

| Property | Recorded result |
| --- | --- |
| Image size | 18,030,592 bytes, unchanged |
| Literal offset / length | 3,305,706 / 34 bytes |
| Changed byte values | 22 |
| Entries inspected | 525 |
| Changed path | `/etc/passwd` only; mode/owner/time/size unchanged |
| Original SHA-256 | `8764ce6b41648d19d3bc386ec1a784d8df27f648afcc7d8cc9be576c7d6b2d91` |
| Candidate SHA-256 | `92273d43ef7bf4e100582a7f2165af7ea809a7955670b09028e039944e4f3dcf` |

Independent extraction accepted the result, still LZ4-HC with 128 KiB blocks,
34 fragments and 525 inodes. The tool refuses missing/nonunique literals or
length changes. It is not a general SquashFS editor, signed update package or
raw NAND programming image. Later offline signature checks confirmed that a
body edit fails authentication even after refreshing the stored digest; see
[stock update authentication](ascent-stock-updater.md).

USB exposed networking and ACM management, not a block device for mounting the
onboard flash. Direct NAND modification would additionally need verified UBI
mapping, geometry, bad blocks and ECC/OOB treatment. It bypasses the updater's
signature gate but does not prove acceptance by every boot-stage integrity check.
The chip-off password proposal was not physically tested on this receiver.

## First deployed wrapper on 17.5.3

On October 5 the test installed a readable continuation before replacing
`/usrdata/run.sh`. The continuation was explicitly selected from official
**17.5.8**, not read back from the receiver's former 17.5.3 startup. That missing
original backup prevents claiming equivalent vendor startup behavior.

| Installed path | Bytes | SHA-256 |
| --- | ---: | --- |
| `/usrdata/run.original.sh` | 7,053 | `359d9365ece77a61e916d36e901bdb424235cf6a161bbf474b376a495c0dc3f8` |
| `/usrdata/run.sh` | 3,719 | `5017893b7c7ea1c204d93a2ccfdd8bbc0068ba5373c94714a10e05db2d0dfa39` |

The wrapper reads the live account list, requires exactly one valid UID-0 root
entry and alters only that credential field. It creates private RAM backing
files and bind-mounts passwd, plus shadow if already present, then sources the
continuation. It preserves other accounts and existing shadow aging policy.
The account update runs in a subshell to isolate traps, variables and umask.
Mount failure attempts reverse cleanup and still permits vendor continuation;
backing files remain if unmount fails. Missing/recursive continuation is refused.

The initial wrapper upload timed out at metadata, before any wrapper data. A
prepared-session retry succeeded with complete byte-count/CRC/sequence ACKs.
FileEnd was omitted, storage writeback was allowed, and the receiver restarted.
Independent authenticated SSH readback matched both script hashes and returned
UID 0. The root filesystem remained read-only SquashFS, `/usrdata` writable UBIFS,
and `/etc/passwd` a tmpfs-backed bind mount. Firmware still reported 17.5.3.
Normal service presence was verified; this was not video/RF qualification.

NetworkManager subsequently removed the manually assigned host USB address,
causing a pre-authentication timeout despite healthy device identity responses.
Runtime-unmanaged host addressing restored SSH. The host used
`192.168.3.222/24` with no default route; interface names changed on enumeration.
An authentication diagnosis must distinguish that host-route failure from a
rejected password.

## Reboot-verified 18.21.10 overlays

After the firmware update, USB identity/networking worked but the earlier
credential was rejected. On October 8 the tested
[clean-mode external-script hook](ascent-startup-hooks.md#clean-mode-external-script)
installed a replacement service using the actual updated vendor script.
The board reported Gnd485, 18.21.10, build `202608141800`.

The installer verified staged checksums, preserved the current startup script,
installed a separate root-readable hash input and explicit startup hook, then
started normal init once and exited the sourcing clean shell. An ordinary reboot
preserved the external-script selector; a vendor `clean` request would have
overwritten it. No FileEnd, firmware image, boot-environment or raw-NAND write
was part of this installation.

Stock SquashFS lacked `/etc/shadow` and the new service pathname. The service
therefore copied the current `/etc` tree with modes preserved into private tmpfs,
bound that directory over `/etc`, then made separate passwd and shadow binds:

```text
SquashFS /etc → private RAM copy → bind over /etc
generated RAM passwd            → bind over /etc/passwd
generated RAM shadow            → bind over /etc/shadow
```

Root's passwd marker becomes `x`; UID/GID/home/shell and other accounts remain.
Existing non-root shadow entries survive, or are generated from the original
account list when absent. Root expiry/change-forcing aging fields are cleared.
Passwd is 0644, shadow/hash 0600 and the service 0755, with private directories.
Every boot rebuilds the RAM tree from the preserved filesystem. Repeated start
validates same-boot state/mounts; partial setup rolls mounts back in reverse order.

| Persistent component | Recorded identity |
| --- | --- |
| Actual 18.21.10 vendor continuation | 6,658 bytes; SHA-256 `a5723f47884216070b6db75e39548db31f711f88c8e121bc6afa6f259fe89c76` |
| Account service | 4,861 bytes; SHA-256 `3ef16ce6394b0f17e5587fcfb1f9b2d0590fa9d762852dfcc8bf1a70fd020597` |
| `/usrdata/run.sh` hook | SHA-256 `0b5057f43e97ac6fb9c5a7260bf38f847dd551d19c99abf54f2f12be91781a5f` |

The hook explicitly calls the service before sourcing the preserved continuation;
its S-style filename alone does not enable startup. It is exposed in RAM as
`/etc/init.d/ascent-access` and backed by persistent
`/usrdata/ascent-access/init.d/S10account-overlays`.

The installation boot was `6d2e4315-7b57-41f6-9fb1-0dc170eb51a3`.
Ordinary boot `47f37630-2e44-4e13-ba0a-148f5bb52f44` recreated all three mounts,
kept installed hashes, authenticated root SSH and restored normal radio/backend/UI
services. The retained installer result still identified the first boot, while
per-boot service state identified the second. Staging, selector and clean-mode
temporary mounts were absent after cleanup.

## Writable-etc side effect and restoration boundary

The vendor gadget script appends `ttyGS0::askfirst:-/bin/sh` to inittab and reloads
init. Original read-only SquashFS rejected that write; RAM `/etc` allowed it.
An init child then competed with the binary updater on ttyGS0, and identity
responses contained console text. The account service does not itself add this
entry. Verified USB Ethernet/SSH does not imply binary management can coexist
with that console.

Restoring `/usrdata/run.sh` from the preserved vendor continuation would disable
the account hook on the next boot and restore vendor authentication behavior.
It would not remove current RAM mounts immediately. That rollback was described
but not executed in the recorded installation. Private credential material is
not needed to understand these mechanisms or their verification.

## Separate manufacturing-service lead

The earlier comparison with `aj22r/walksnail-root` at revision
`7f5a809c77f97522bb341c253b66c43fce72a8e2` did not establish an applicable
Gnd485 root route. That code targets Goggles X's memory-debug service on TCP
1234, using an `ff 5a` header, a different DRAM range and persistence under
`/local`. Its expected command-string marker was absent from the selected
Ascent binaries. Changing the address or substituting radio port 50000 would
not adapt the protocol. The inspected script also lacked a stop when its marker
search failed; its default zero offset could still reach the write step.
No part of that exploit was run against this receiver.

Ascent's reference `ar_fpv_mp_service` instead listens on TCP 6835 and checks
`fa fa` magic in a 16-byte header, with payload length at offset 6 and payload
checksum at offset 14. A bounded live connect/banner check found 6835 open
without a banner, alongside SSH and radio TCP 50000; 1234 was not observed open.
No application request or memory operation was sent to 6835 in that test.
The matching port is a concrete service lead, not proof of a memory-access
protocol, command execution or another password bypass. Later startup-overlay
access does not validate this separate route.
