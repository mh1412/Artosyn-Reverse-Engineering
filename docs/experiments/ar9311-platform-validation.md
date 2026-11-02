# Gnd485 platform validation and recovery

[Experiments](README.md) · [Platform registers](../soc/ar9311-platform.md)

The October 2026 AR9311 work first established a USB RAM-booted Gnd485 baseline,
then progressed to installed Linux 6.18.36 with a separate clean recovery image.
Earlier RAM-only statements describe those early tests, not the final boot scope.
This page records platform acceptance and failure boundaries; it does not claim
every AR9311 peripheral or production use case is complete.

## Clock ownership control experiment

Known-good kernel/DT/initramfs images were uploaded and read-verified through
U-Boot USB, with independent UART CRC checks. A control deliberately cleared only
the four MMC gate masks before boot. The old image timed out resetting both MMC
blocks: neither SD nor radio appeared, while Linux, USB and decoder still ran.

The new source-clock provider recovered from the same initial fault, restoring
the baseline gate words while preserving unrelated fields. An 8 MiB SD
write/flush/cache-drop/reread matched SHA-256
`98e08ed5596890789e3ce13dbedfa408a13161782ea63827be5a815edac2bdfd`.
Radio enumerated as `4152:8031` after one upload. Its short reception test passed
ten control acknowledgements and transferred 8,898,041 bytes in 15.002 seconds,
with zero sampled socket drops and bus errors. Complete stream decoding was not
established by those counters.

Unmounting and unbinding only the removable SD host dropped its clock count to
zero while SDIO remained enabled. Rebinding restored the SD hash and allowed
radio reopen without another firmware upload. This tests isolation between
those two consumers, not general SoC clock management.

## GPIO handover and cold boot

Direction preservation retained the bootloader's SD-power output and matched
configured directions after Linux consumers registered. A cold boot reached two
CPUs, RAM root, pstore, SD, radio, decoder and DRM with no deferred devices.
Kernel taint 4096 represented out-of-tree modules. The direction measurements
did not include oscilloscope verification of transient glitches.

One 16 MiB memory pass and concurrent CPU-pinned 8 MiB tests completed. They are
bounded memory/CPU checks, not full RAM, thermal or environmental qualification.
USB had 30/30 steady ping replies; intentional gadget restart lost one ping,
then re-enumerated and restored SSH without changing the SD checksum or radio
upload count.

## Media and process recovery

Ten H.264 720p and ten HEVC 1080p sessions, eight frames each, reproduced exact
reference hashes. The first test invocation used incorrect fixture paths and
failed before decoding; the corrected run supplied the actual evidence.

Killing a streaming player with SIGKILL was followed by a successful 600-frame
1080p60 DMA playback at 59.909 fps, zero late frames and zero CPU pixel copies.
Only that fixed-resolution HEVC kill/restart case was covered. Forced scanout
and flip counts with a disconnected sink do not independently prove external
HDMI picture quality.

## Reset and crash evidence

Warm software restart reached SPL/U-Boot and same-image USB recovery, retaining
unique console/pmsg markers. A SysRq panic after SD unmount restarted after five
seconds and preserved the panic trace. An explicit watchdog client stopped
feeding with its descriptor open and caused hardware reset without a power cycle.
Recovered records were archived to SD and checksum-verified.

Final cold boot retained the archived SD records and checksums, while pstore
itself was empty as expected. Magic close and inactive software status did not
stop the hardware watchdog. The RTC candidate remained nonfunctional and was
reverted; these limits are detailed in the [platform study](../soc/ar9311-platform.md).

## Later installed-platform regression

The return to Linux 6.18.36 validated initial NAND install, normal boot into clean
recovery for self-update, exact-image boot verification and unattended cold boot.
The optional stock-kernel profile remained a separate compatibility experiment.
Shared kernel/DTB writes were not power-loss atomic, and no new automatic A/B
health rollback was established.

The later combined regression tested runtime source `5222203` with bundle
SHA-256 `cde6ca8a01f6171a182dd32d2b44827538b8adb6dfce724822da928922658181`:

| Area | Result |
| --- | --- |
| Host checks | 75 updater tests, ten extractor tests, six radio recipe cases, three board configurations and 13 build routes |
| Upload/retry | Malformed and oversized upload rejection, valid retry, exact-image NAND boot |
| Platform/storage | Zero checker failures, 16 MiB memory test, SD fsync/readback and persistence through update/power cycle |
| Decode | Three H.264/HEVC reference-pixel rounds plus a cold-boot repeat |
| Scanout | Both unforced players: 600 frames, 59.909 fps, zero late frames and zero CPU pixel copies |
| Radio | 13,865,244 bytes in 20 seconds, ten acknowledgements, zero SDIO errors and no extracted bad headers/tails/XOR checks |
| Update/recovery | Clean-mode self-update with automatic exact-image verification and cold boot |
| Vendor restoration | Original kernel/DTB hashes restored; protected, stock-root and vendor-data regions unchanged; zero uncorrectable ECC failures |

Stock 18.21.10 USB identity and UART GUI startup were confirmed at the end.
Other board results in this batch were configuration/recipe checks, not physical
validation. Destructive update fault cases remained simulated. Capture-side
black intervals had an unresolved cause and were outside that batch's acceptance
criteria. USB connected automatically during the run, but historical intermittency
was not declared fixed. There was no loss-free end-to-end radio video claim.

These later results supersede early “RAM boot only” and “NAND disabled” milestones.
They do not supersede unresolved watchdog stop, RTC, broader media recovery,
HDMI hotplug/colorimetry or complete RF/application acceptance.
