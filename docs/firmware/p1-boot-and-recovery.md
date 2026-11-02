# P1 direct NAND boot, update and recovery

[Firmware](README.md) · [P1 Air](../devices/betafpv-p1-air.md)

Scope: the P1_SKY 1.0.44 partition layout and the direct-boot development image
validated by 2026-11-02 00:04 UTC. This is a technical account of the tested
design, not a model-independent flashing recipe.

## Progression of boot methods

The installed vendor Linux lacked both kexec entry points. Early source-kernel
tests therefore used an exact-build RAM handoff module and ARM64 trampoline,
with explicit image/DT/initramfs bounds, readback verification and secondary-CPU
parking. A cold power cycle restored stock firmware. That bench loader was not
a generic kexec implementation.

A later stock-stage installation stored a source bundle in the alternate
application volume and used a boot agent. It was superseded by **direct source
Linux boot through the original SPL**, without a stock userspace stage for
normal operation or native updates.

The final design keeps stock bank 1 and installs fpvOS in bank 0. It is
stock/fpvOS selection, not two interchangeable fpvOS rollback slots.

## Verified partition boundaries

The source board definition exposes only the needed update/state regions and
marks factory and the reference stock kernel read-only:

| Region | Offset | Size | Purpose |
| --- | --- | --- | --- |
| Boot state | `0x100000` | `0x80000` | Redundant GPT selection records |
| Factory | `0x200000` | `0x600000` | Read-only calibration |
| Kernel 0 | `0xa40000` | `0x600000` | Source kernel and embedded recovery |
| Stock kernel | `0x1040000` | `0x600000` | Preserved bank-1 reference |
| DTB 0 | `0x1640000` | `0x60000` | Source board description |
| User application 0 | `0x1700000` | `0x2d00000` | Existing UBI volume holding source bundle |
| User data | `0x7100000` | `0x600000` | Existing persistent application settings |

These are the exposed regions, not a complete inventory of every stock
partition. SPL, U-Boot, environment and the preserved stock bank are not rewritten
by the source installation procedure. No format, resize or repartition is used.

## Installation transaction

1. Validate the exact layout and input bundle.
2. Select preserved stock bank 1 before changing the source bank.
3. Write and verify kernel 0 and DTB 0, accounting for the supported NAND geometry
   and bad-block behavior.
4. Update the existing `userapp0` UBI volume with the complete `FPVOSUP1` bundle
   and verify its contents.
5. Activate all three bank-0 components together only after successful readback.

Activation erases the nonselected GPT copy, advances its generation, recomputes
CRCs and verifies the new record. A torn record must leave the preceding valid
selection available. Fault-injection tests exercised write/readback failures,
bad blocks, torn records and retry/activation ordering.

The original stock clean hook is used only as an initial installer entry point.
The temporary bootstrap removes its staging files; normal source operation and
updates do not depend on a persistent stock-root patch.

## Independent recovery

The source kernel embeds recovery userspace with USB Ethernet, SSH and CDC-ACM
update service. Recovery validates the stored bundle and matches its kernel/DTB
against NAND before unpacking the normal root into RAM. Damaged, missing or
mismatched bundles remain in recovery. A clean-mode request is consumed once.

The ARM64 entry guard arms the watchdog and a volatile SPL USB request at
`0x0a106138`. A tested request value of 3 enters the U-Boot USB downloader;
normal reboot uses 2 to clear request bit 0. Zero is unsuitable because the
reboot-mode implementation ignores it.

In the recovery test, stopping watchdog feeding returned the device to U-Boot
USB after 90 seconds. USB RAM rescue then restored source Linux without another
flash write. This does not cover damaged SPL/U-Boot or failure before the kernel
entry guard executes. The rescue helper used in that experiment was a lab tool,
not a demonstrated general consumer recovery application.

## Boot dependencies found during the work

Direct boot exposed an inherited QSPI clock dependency: U-Boot's 150 MHz source
caused shifted NAND identification. An independent clock provider using 25 MHz
and reset sample delay fixed detection while preserving adjacent I2S fields.
Starting from vendor Linux had previously hidden that dependency.

The tested U-Boot path rejected legacy LZ4 and accepted a standard LZ4 frame
with independent blocks, carried in the expected OTRA/uImage form. RAM load and
entry for this path were `0x20200000`.

Raw MTD writes were synchronous but did not implement fsync. An initial fsync
error stopped installation after a kernel block while stock remained selected.
The corrected installer and its fault harness model that MTD behavior explicitly.

## Validation and artifact identity

The final test sequence covered initial installation, direct boot, native
self-update, rejected corrupted upload, two full power cycles, stock selection
and the watchdog rescue. Twelve protected partitions matched the verified
backup on stock return. Calibration, settings and peer hashes were preserved.

One cold-boot capture contained 1,073 complete HEVC frames and 8,198,759 encoded
bytes, with all ten control acknowledgements and no invalid headers, tails,
XOR checks or skipped bytes. All complete frames decoded successfully. Capture
ended inside the next packet; its 4,532-byte incomplete tail was excluded.

| Artifact | Bytes | SHA-256 |
| --- | ---: | --- |
| `fpvos-p1-air.fwu` | 8,262,203 | `6cd904bffa57db890a41f401e12e089cefc9b507ee5c00cd166bfdcad4c83023` |
| `kernel.img` | 5,294,080 | `361736b3949ac03b87b43926508768a071da182f61543cc51ccd9f8ff2785a1c` |
| `betafpv-p1-air.dtb` | 8,940 | `e701af012e2b6d6481359abed266413ac39ac5df04e5b6d3f192c5320814109c` |

Ninety host updater tests and the P1/shared Ascent builds passed in the recorded
work. There was no deliberate power interruption during a real flash write,
and no Ascent hardware regression test in this P1 milestone. Bundles were
unsigned development images without version-order enforcement; source Kestrel
was supplied through a development checkout rather than a published package pin.

## Source publication checkpoint

The development record identifies fpvOS commit `e32ea69` and companion
Kestrel Air commit `d33b6e7`. Its November 2, 2026 publication entry records
draft submissions [fpvOS #10](https://github.com/gehee/fpvOS/pull/10) and
[Kestrel Air #2](https://github.com/gehee/kestrel-air/pull/2), recreated after
renaming the source branches to `p1-air-support`. The earlier #9/#1 submissions
were closed during that rename; the recorded code hashes did not change.
This identifies the submission checkpoint, not a claim of upstream merge or
current PR status.

The recorded source audit found no committed vendor images, backups or build
artifacts in the reviewed changes. Base radio configuration cleared candidate
peers and per-device offsets and used identity placeholders resolved at runtime;
actual calibration remained a factory provisioning input. The audit combined
pattern scans and manual match review, not a guarantee against every possible
embedded secret. Publication did not resolve the companion-package pin dependency
or broaden the hardware qualification above.
