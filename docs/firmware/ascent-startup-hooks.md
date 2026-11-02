# Ascent stock startup and one-shot maintenance hooks

[Firmware](README.md) · [Gnd485](../devices/ascent-vrx.md) · [Account overlays](ascent-root-access.md)

The October 6 investigation examined live 17.5.3 rootfs init/update files on
Gnd485. Its normal boot already included the lab password wrapper and selected
17.5.8 continuation. The stock paths are explicit callers; no general writable
directory or SD-root script scanner was found in the inspected implementation.

## Normal startup

BusyBox init runs `rcS`, which calls optional `start_nodev.sh`, `mdev -s`, optional
`start_fast.sh`, `mount_fs.sh`, then either clean startup or `start.sh`. The
comment claiming numerical init-script order does not match a general S-script
scan in that code. Optional fast scripts were absent in the captured system.

Normal `start.sh` sources `/usrdata/run.sh` asynchronously. Radio/backend scripts
and the GUI loop are called by name downstream. `/usrdata/run.original.sh` was
a lab-added continuation, not an independent stock hook. The named
`/tmp/fpvdata/bin/GlassesUI` is on persistent UBIFS despite its `/tmp` prefix;
ordinary `/tmp` is RAM-backed and disappears at reboot.

New USB-uploaded files were mode 0644. Sourcing needs readability; direct execution
needs execute permission. The uploader does not automatically chmod or run text.
Writable PATH/library directories affect requested names, not arbitrary discovery.
For example, the existing rootfs `/usr/bin/mpp_service.app` resolves before a
new similarly named writable-directory file.

## Clean-mode external script

The live selector is `/factory/sirius-clean-system-flag`. For a nondefault mode,
`start_clean_system.sh` extracts the second colon-separated field, removes the
selector, syncs, then sources that pathname. The tested shape was:

```text
web:/usrdata/selected-maintenance-script.sh
```

The `web` label chooses this branch; it does not create a web server. The script
must already be readable when mounted startup reaches it. Space/colon-containing
paths are not robustly parsed. A malformed `ssh` condition in the original shell
code prevents generalizing the verified `web:path` behavior to every mode label.

The path is consumed before execution, making the selector one-shot unless
something recreates it. PID 1 retained a legacy cmdline flag path under `/usrdata`,
but `rcS` overwrote the child environment with `/factory`. The separate
`clean-system-enter.sh` derives its path from cmdline and therefore cannot be
assumed to select the same file.

A normal reboot preserves a prepared selector. The serial `0x03` request with
payload `clean` instead writes `usb` into it, replacing an external-script path.
USB session preparation `0x72` is a third, separate operation. Neither command
should be equated with the updater's OTA preparation hook below.

Clean startup skips normal `start.sh` and the password wrapper. If an external
script simply returns, the remaining clean/update setup continues. The tested
maintenance script deliberately launched normal `start.sh` once and **exited the
sourcing shell**, rather than returning. That recovered normal services; it is
not automatic stock behavior for every external script.

## Physical one-shot execution proof

An initial harmless 40-byte factory-directory upload returned complete valid
ACKs and independently matched SHA-256 over SSH, then was deleted without arming
the selector. The later controlled execution test uploaded the actual selector
and a 2,350-byte script with mode 0644, independently checking both hashes before
ordinary reboot. No FileEnd or firmware image was sent.

| Observation | Evidence |
| --- | --- |
| Selected boot | `78d4d11d-d693-4e53-ac56-a7cab7654e19` |
| Execution identity | UID 0, mode `web`, factory selector already absent |
| Sourcing caller | `/bin/sh /etc/init.d/start_clean_system.sh` |
| Execution time | Uptime 2.37 seconds; wall clock still January 1970 |
| Downloaded marker | 313 bytes; SHA-256 `e6eb29a7467ef4e3ba67d43e2df25a2221c3a32bebb1321eb36a0ef9d0166675` |
| Second ordinary boot | `736215bf-5b4a-4825-a0c9-14f0eab75b09`, marker unchanged, selector absent |

The unchanged marker includes the earlier boot identity and normal-start PID;
it demonstrates one-shot behavior rather than merely successful later access.
Root SSH, radio/backend/UI and ordinary updater arguments returned. The previous
factory upgrade-mode contents `1\n` were recreated unchanged. Clean-mode staging
mounts disappeared after the second boot, and test script/marker were removed
after evidence capture. Firmware remained 17.5.3.

The initial text-log inspection stopped on invalid UTF-8. Subsequent complete
verification and binary download supplied the proof; the partial transcript was
not treated as success. Host tests exercised sourced-shell exit, failed marker
writes and invalid invocation with simulated commands, separately from this
physical execution.

## Temporary OTA preparation override

The captured live updater SHA-256 was
`29f929d08cbfede276bbaec2d7ef4f091eba1f31c69801cc00ef38848a08eea2`.
At `ar_upgrade_prepare` address `0x42c840`, code checks for
`/tmp/ota-prepare.sh`, otherwise selects `/etc/ota-prepare.sh`. `vsystem` invokes
the bare path through `/bin/sh -c`; this is direct execution, requiring execute
permission even though the existence test uses only `F_OK`.

This is a binary-confirmed update-event hook, not a startup watcher, and was not
executed during this investigation. The stock fallback does work only with
argument `real_do`, while this call supplies no argument. That distinction and
the separate update transaction are covered in [stock updater analysis](ascent-stock-updater.md).

## Other mechanisms examined

| Candidate | Observed behavior |
| --- | --- |
| SD hotplug | mdev insert mounts vfat at `/mnt/sdcard`; remove unmounts; no script scan in those handlers |
| Recording card | Explicit mount of `/dev/mmcblk2p1` at `/usrdata/record`; data use, not shell autorun |
| Cron/run-parts | Tools exist, but no active scheduler/configuration or boot caller was observed |
| rc.local/profile.d/hotplug.d | No relevant caller/directory in inspected startup paths |
| Login profile | Environment exports; shell-session behavior, not general boot autorun |
| Network upgrade staging | Conditional legacy chroot/script path with absent assets; not ordinary startup |
| ADB event | Handles USB state and returns without `ffs.adb`; not card execution |

Searches covered live shell paths and strings in the exact application, media,
updater/backend/manufacturing binaries. String absence cannot rule out dynamically
constructed paths or unexamined components. No SD insertion/script test was done.
The recorded negative result is therefore scoped to inspected paths, while
normal named startup and `web:path` execution have direct evidence.
