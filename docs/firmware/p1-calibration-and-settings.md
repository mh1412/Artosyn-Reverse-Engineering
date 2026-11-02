# P1 calibration, settings and application startup

[Firmware](README.md) · [P1 Air](../devices/betafpv-p1-air.md)

## Calibration is per-unit data

P1 uses host NAND UBIFS factory data, including `/factory/ar8030_pwr.json`, as
the authoritative power-offset source. A generic firmware download cannot
replace that unit's calibration.

This differs from the investigated Ascent VRX, whose radio reported a separate
2 MiB SPI flash, factory record at offset `0xa000`, and version-4 offsets selected
by the radio itself. The P1 radio-side factory attempt returned `-19`; its
validated source path supplies the host factory offsets instead.

The source P1 configuration preserves OTP as the identity source. It extracts
board defaults from pinned 1.0.44 firmware, adapts the validated AP settings,
and merges this unit's offsets at boot. Stock merged JSON can contain DEV-role
defaults that the vendor application changes later, so it cannot be passed
unchanged to the replacement AP firmware.

## Read-only factory mount

Factory occupies 6 MiB at NAND offset `0x200000`. Startup validates the
partition name, position, geometry and protection, attaches UBI read-only and
mounts the `factory` volume with `ro,nodev,nosuid,noexec`.

There is no format, repair, resize or fallback calibration operation. Failure
leaves management access available and stops dependent radio startup. Hardware
tests checked full-partition hashes before and after attach, reattach and video,
as well as rejection of writable MTD/file access.

Stock Linux can update UBI metadata during its own mount. Byte preservation
across a source session therefore uses its immediately preceding baseline;
calibration-file hashes provide an additional cross-boot check.

## Validated merge rules

The calibration helper accepts only the nested schema
`baseband.basic.power.pwr_offset`, containing exactly four integers from −6 to +6.
Unknown or duplicate keys, malformed/oversized JSON, invalid values and
input/output aliasing are rejected.

The helper updates only the offsets and atomically writes a mode-0600 runtime
configuration in RAM. Failed provisioning must not load an older configuration
left behind by an earlier run. An already running radio remains untouched;
changed calibration is applied on a cold boot.

The tested radio reported OTP identity and JSON power source, with the runtime
JSON exactly matching board defaults plus its own factory offsets. Video and
command checks passed. This verifies data selection and transport behavior,
not calibrated RF output-power accuracy.

## Persistent user settings

The existing 6 MiB `usr_data` partition at `0x7100000` remains separate from
factory. Source startup mounts its existing UBIFS volume at `/persist` and
binds a dedicated `fpvos/kestrel-air` directory to `/var/lib/kestrel-air`.
Vendor `lowdelay` files remain intact. No formatting or filesystem conversion
is required, and Kestrel refuses a missing writable mount rather than silently
saving to RAM.

`settings.json` and `peer.json` use a complete temporary write, file fsync,
rename and directory fsync. New files use mode 0600; symlink and special-file
destinations are rejected. Failed durability checks propagate to the caller.
An abandoned temporary file is not treated as active settings.

The persistence experiment saved a preference and receiver peer, removed power
after the writer returned, then read identical bytes from stock Linux and a
subsequent source boot. Video resumed without re-pairing. This is not an
exhaustive power-cut test at every NAND write boundary.

## Autostart and supervision

Startup orders factory mount, radio calibration/loading, settings mount and
Kestrel. The application and AE/AWB helpers share a process group supervised
with subreaper cleanup. A kernel file lock excludes simultaneous recorder use.

Exited applications restart after two seconds. Five runs shorter than 60 seconds
exhaust the retry budget; a stable run resets it. Normal stop allows five seconds
for graceful cleanup, then escalates and bounds further cleanup before permitting
another camera owner. Two RAM logs are capped at 1 MiB each.

Hardware tests covered cold autostart, active Kestrel and AE-worker termination,
helper cleanup, missing-storage exhaustion and recorder handoff. Receiver setup
was repeated after crashes to restore video. Uninterrupted playback across an
application crash and recovery from kernel-stuck workers were not demonstrated.
