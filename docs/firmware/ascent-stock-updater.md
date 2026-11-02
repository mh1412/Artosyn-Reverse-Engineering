# Ascent stock update engine and authentication

[Firmware](README.md) · [USB protocol](../protocols/ascent-usb-management.md) · [NAND layout](gnd485-nand-and-gpt.md)

The Gnd485 `ar_fpv_upgrade` program is a resident management service with a linked
image/flash engine. It is not just a wrapper around `artosyn_upgrade`. This
account separates static traces from live transfer/boot tests and uses the exact
18.21.10 running executable for current workflow claims.

| Examined updater | Bytes | SHA-256 |
| --- | ---: | --- |
| Live 17.5.3 | 915,880 | `29f929d08cbfede276bbaec2d7ef4f091eba1f31c69801cc00ef38848a08eea2` |
| Reference 17.5.8 | — | `ed954d62acfe834b277c513eb0ec7e511ff775aea5e1707dbd072b2eb9255650` |
| Live 18.21.10 | 956,888 | `c414bce817b9ada660ab884ca1e860603d344025abe1ba9e61bceaf8dcb2f4b3` |

## Normal and clean operation

Normal startup copies the updater to RAM and runs it with `-c 1 -b 0 -r 16`.
Despite the field name `is_cleanmode`, `-c 1` is this ordinary mode; `-c 0`
takes the worker's stop/stage/unmount path. `-m 0` means SD source, `-m 1` PC
receiving; `-i` starts an immediate upgrade and `-v` enables progress output.
These options differ from the standalone updater CLI.

An applicable `0x03` request containing `clean` writes `usb` to
`/factory/sirius-clean-system-flag`, syncs and invokes a watchdog reboot helper.
The next stock Linux boot consumes that flag and stages a partial runtime in
RAM. It uses the same NAND-loaded kernel, not a separate recovery kernel or
U-Boot download mode.

The script copies selected executables/libraries, device nodes, factory data and
display assets, consumes the update-mode flag, runs
`/etc/ota-prepare.sh real_do`, unmounts persistent data mounts and remounts root
read-only. It recreates RNDIS/ACM and launches the staged updater. A separate
`web:path` branch consumes the selector and sources an external shell script;
later custom bootstrap work uses that hook.

The prepare library prefers a temporary `ota-prepare.sh` override but invokes
the bare pathname. The stock script returns success without doing preparation
unless passed `real_do`. A successful library call alone therefore does not
prove mounts were released. The similarly named `ar9301-upgrade.sh` in 18.21.10
also prepares mounts; it is not the separate generic `artosyn-upgrade.sh` raw
flash writer.

## Worker selection and staging

Ground board types 1/5 enter the ground worker; Gnd485 is type 5. PC mode scans
staging for `Ascent_G_Gnd` or `Ascent_VRX` image families. SD mode detects the
recorded card nodes, scans and normalizes a matching image into RAM. Filename
SDK/application/QA comparisons and board/custom-product filters precede the
lower image engine.

The SD normalization routine has a static defect: it can strip the 128-byte ASW
prefix but write the original length, appending unspecified bytes beyond the
valid inner content. Its caller ignores the returned error. This was traced,
not induced in a physical SD update. Other unchecked script results and coarse
worker errors also limit what a success-looking log proves.

After receiving FileEnd, the worker's preparation requires at least 10 MiB
available memory. Images larger than 10 MiB use an initial buffer and streaming
authentication; smaller ones use the buffer path. The clean updater itself is
launched without chroot. Only the final reboot helper runs inside the staged
root.

## Signed package layout

For the inspected 17.5.8 package:

| Region | Absolute offset | Size |
| --- | --- | --- |
| ASW wrapper | `0x000` | 128 |
| Modern OTRA header | `0x080` | 256 |
| Stored SHA-256 | `0x180` | 32 |
| RSA-2048 signature | `0x1a0` | 256 |
| Authenticated body | `0x2a0` | 49,206,986 bytes through EOF |

The body digest is
`8c2a4e585c3f17556baa083f2bc26d6c775e7728555e6496d9e5d7393ecdc72c`.
Offline verification with the live public key matched the stored hash and
RSA-recovered digest tail. The traced verifier is not claimed to perform strict
PKCS#1 padding validation. The wrapper and preceding header are outside this
body digest; no usable malformed-header bypass was demonstrated.

The single trusted key pathname ends in `.pem`, but its 294 bytes are a DER
SubjectPublicKeyInfo object. Key SHA-256 is
`8a6557918855a74d60d1ff60ca437859344700800a9155815e200aa9788363a6`.
The traced loader imports it once without a directory scan, key list or fallback
key. Concatenating another key is not an established multi-key policy.

Changing bytes inside the authenticated body fails the digest/signature checks.
Refreshing the stored SHA makes that hash comparison pass but leaves the old
signature invalid. File CRC/MD5, renaming and root access do not turn such a
mutation into an authenticated vendor package.

Seven runtime policy bytes were observed enabled in 18.21.10: rollback, CRC,
upgrade flags, chip ID, custom product, board type and version. Application
rollback-policy branches are distinct from the lower SHA/RSA path. No traced
runtime flag disabled RSA. DDR checks also have explicit skip cases for unknown
old metadata or unrecognized new FDT; a `force` DDR option is not a demonstrated
signature bypass.

## Flash order and rollback boundary

The NAND path uses linked libmtd/libubi and formatting code through kernel
interfaces. It does not shell out to `flash_erase` or `ubiformat` for its normal
signed-package writes. OTA chunks use LZO1X; LZ4 mentioned by extraction tools
belongs to nested SquashFS.

After authentication and structural checks, the engine attempts four SPL copies,
then selected partition segments and finally GPT commit. The inspected 17.5.8
package order is vendor, BL31, environment, U-Boot, kernel, DTB, userapp, user
data and FPV data. Factory has write flag zero in that package. Paired targets
can redirect to the adjacent member according to package flags and current
GPT bit 47. These destinations are package/state-dependent static predictions,
not an executed update of every listed partition.

The 18.21.10 partition-removal selector excludes the master, factory and
userapp regions. The GPT writer writes the opposite record, then may erase the
previous record in its development-mode branch. Two reserved GPT areas do not
guarantee two usable fallback records afterward.

SPL-copy code attempts each write without checking its result before continuing.
Worker failure can collapse a more specific result to error `-10`; after an
image-engine failure it waits 300 seconds before cleanup/reboot. No automatic
restoration of already-written shared kernel/DTB was traced. The five-boot
[rollback experiment](gnd485-nand-and-gpt.md#restoration-and-rollback-findings)
later found no automatic root-bank switch for early-init panic failures.

## Dependencies and interpretation

The 18.21.10 updater is dynamic AArch64/glibc 2.25 with 61 direct library
dependencies. They include the loader/libc/pthread, Artosyn HAL/MPI/MPP media
stack, IPC/RPC, FFmpeg, audio and C++ libraries. Disabling video at runtime does
not remove the ELF loader's need to resolve them. Crypto, LZO, JSON, MTD and UBI
implementations are embedded, while shell scripts manage mounts, USB and reboot.

Partial RAM staging does not copy every updater dependency. Most libraries can
still resolve from the old root because the updater is not chrooted. The final
watchdog helper has a much smaller explicitly staged dependency set and uses
`/dev/watchdog0` timeout/keepalive ioctls. Copying the main executable alone is
therefore insufficient for an independent recovery environment.

The described October 8 workflow study did not trigger a full vendor update.
Later custom installation and stock restoration results use a different
[fpvOS engine](gnd485-fpvos-update.md). Their success does not validate every
stock error path or establish universal Walksnail/Ascent model compatibility.
