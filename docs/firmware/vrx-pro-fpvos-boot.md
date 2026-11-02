# VRX Pro fpvOS boot, kernel and pairing import

[Firmware](README.md) · [VRX Pro application stack](../systems/vrx-pro-fpvos.md)

This study covers the RK3568 Ascent VRX Pro fpvOS snapshot retained with the
October 2026 research. It combines its kernel/build documentation with inspection
of the boot-chain builder, board-driver patch, pairing importer and packaging
recipes. It does not describe the AR9311 Gnd485 NAND installation or establish
a fresh hardware boot test.

## SD boot and the vendor boundary

The image assembles three initial SD partitions without loop mounts: FAT fallback,
ext4 root, and the FIT boot partition used by U-Boot. First boot expands the
partition table and creates an exFAT recordings partition, sometimes requiring
an automatic restart. Reflashing the card replaces its settings and recordings.

The boot-chain builder accepts two identified stock U-Boot payloads:

| Stock label | U-Boot payload SHA-256 |
| --- | --- |
| 18.21.10 download | `21e604ba916b603b70b1043af895c967edca9425c3bd442b37f46b1607988b05` |
| 18.21.7 | `0548dd678e8f91e298bc79d0098ef6464265c881c82457bab4b3b14586f39be1` |

It processes both 2-MiB copies in the stock 4-MiB U-Boot image, checking each
payload's FIT hash, whole-payload identity and expected instruction bytes before
replacing three hooks. Offsets are within those U-Boot payloads, not flash offsets:

| Offset | Interpreted hook | Replacement return |
| --- | --- | --- |
| `0x3ef4` | Boot beep | 0 |
| `0x3fbc` | Vendor-storage Ethernet address setup | 0 |
| `0x67d0` | Vendor-storage initialization | −1 |

The builder updates the payload hashes. These checks identify known inputs and
detect corruption; they are not signature verification or acceptance of arbitrary
vendor versions. ARM Trusted Firmware and OP-TEE remain stock-derived components.

The kernel path can use stock or an explicitly supplied source-built arm64 Image.
The supplied image is LZ4-framed with a four-byte little-endian decompressed-size
trailer. The same kernel goes into the main FIT and fallback. Decompression must
match the trailer's length; a supplied Image without a readable Linux version
banner is rejected.

The device tree is updated in both the FIT and the Rockchip resource bundle.
The latter is therefore **not byte-identical to stock**: its `rk-kernel.dtb`
entry, length and hash change while other entries remain in place. Replacement
must fit the existing allocation. SHA-1, SHA-256 and unhashed entries are handled
according to the recorded hash-size field; the rebuilt FIT uses SHA-256 component
hashes and the original timestamp. These are packaging integrity fields, not a
new authenticated boot chain.

## Source kernel and device-tree changes

The examined kernel pin is Rockchip `develop-6.12`, commit
`470f9dccbdc42e7b8a824d0a5c5640a10e9457d2`, identified as Linux 6.12.69.
Its archive SHA-256 is
`3b6da867142d24b7d0282763debc072992f5d3bccc8c3dd7c9e68186b0284990`.
This is a historical build identity, not a current kernel recommendation.

The configuration carries forward the extracted stock 5.10 configuration with
board-specific changes. It selects Panfrost and RTL8XXXU, restores the renamed
Rockchip OTP option, and omits kbase, unused camera/tuner drivers, the absent
IT6161 bridge and the failing HDMI-CEC path. Patches address an unconditional
external-PHY dereference, board GPIO/fan services, streamed video decoding and
two Wi-Fi AP issues: group-key loss when a station leaves and buffering for
sleeping stations.

| Device-tree change | Purpose and scope |
| --- | --- |
| SD root arguments | Mount the card's second partition as ext4; separate fallback uses 3.3-V, 50-MHz SD settings |
| VOP plane allocation | Move Esmart1 to HDMI video port 0: masks `0x15 → 0x1d` and `0x2a → 0x22`, leaving Smart1 as port 1's primary |
| GPU names and OPP table | Lowercase interrupt names, `gpu`/`bus` clocks, remove two alternate-bin entries and six unsupported bin masks |
| USB PHY clock names/parent | Adapt the stock tree to the newer USB480M mux naming so the gadget PHY can probe |
| GIC/ITS coherency | Add `dma-noncoherent` to avoid the recorded ITS command timeouts |

The actual builder applies the USB-clock and GIC edits together for supplied
kernels at least 6.10. Its commentary traces the GIC requirement to an earlier
kernel change, but it does not independently gate that edit at 6.5. All expected
tree substitutions are checked. Two NV12-capable display planes support the
application's split-picture path; they do not prove resumable Wave521C decoding
on AR9311.

## Reconstructed board services

The replacement driver binds the stock `rp_power`, `rp_gpio` and
`fan_gpio_control` nodes. The retained board mapping is:

| Signal | GPIO | Active behavior |
| --- | --- | --- |
| USB power / hub reset | 0_A6 / 0_B0 | High |
| SD power / OTG mode | 0_A5 / 1_A4 | Low |
| Speaker enable / mute | 1_B1 / 1_B2 | High / low |
| Status LED | 0_D6 | 500-ms toggles, nominal 1-Hz cycle |
| Additional board lines | 3_B5 / 3_A7 | High |
| Fan | 3_B6 | Thermal policy below |

GPIO function codes 0/4 drive the declared active level, 1 selects input, and
3 blinks. The input path does not reproduce vendor key-event reporting; function
4 does not recreate the vendor userspace switching file. Individual unusable
lines can be warned about and skipped, so a successful probe alone is not proof
that every line was configured.

The fan polls once per second. Defaults turn it on at 60°C and off only below
55°C after at least ten seconds running; device-tree properties can override
threshold and minimum run time. Missing or unreadable temperature turns it on.
This polling implementation avoids reliance on the thermal zone's power-model
governor. Source inspection establishes policy, not measured temperature accuracy
or cooling performance.

## Pairing import and the NAND qualification

The card's `/factory/user_cfg.json` stores a ring of up to 100 air-unit identities.
At boot the importer exposes the stock factory range at offset `0xe00000`, size
16 MiB, using 128-KiB eraseblocks. It requires the `UBI#` header at every block
before attaching UBI and mounting volume 0 read-only. It does not invoke the
vendor mount script that formats after a mount failure.

With no card list, the stock file is copied. Otherwise valid four-byte identities
are deduplicated and added to empty card slots from the saved position, wrapping
around without evicting existing peers. A full ring skips additional identities.
New bindings remain on the card; this code does not write them back to stock.
The rewritten merge serializes its recognized ring, position and power fields,
not arbitrary fields from the original JSON. The initial unmodified copy has a
different preservation boundary.

**Read-only UBIFS does not prove zero NAND writes.** The script attaches UBI
normally, and explicitly acknowledges possible UBI maintenance such as scrubbing
a block with bitflips. Its checks avoid deliberate reformatting and application
writes to the stock list; they do not establish that the underlying attachment
can never program flash. The broad user-guide claim that fpvOS never writes
internal memory must be qualified accordingly. Card removal restores selection
of the stock boot path, not proof that no storage maintenance occurred.

## Packaging and checks performed here

The post-build hook prevents extracted MPP from overwriting the source-built
library, omits libmali when Mesa is selected, and drops the vendor radio
daemon/driver/client when ar_libre is present. Vendor kernel modules with a
different release string are omitted. Equality of that string alone is not a
complete ABI check. The stock updater and formatting factory-mount script are
also removed. Obsolete overlay destinations are removed before copying, avoiding
accidental writes through existing symlinks.

Host shader-build packages are pruned from the target, while the required
`libSPIRV-Tools.so` is retained. The hook checks remaining ELF dependencies
against removed library names. That is not a universal runtime dependency test.
An embedded pairing list produces a warning; the hook does not itself reject
the build. Release handling must distinguish generic images from per-device data.

During this documentation review, isolated synthetic checks confirmed merge
deduplication, wrap into empty slots, full-ring non-eviction and supported-field
serialization. Synthetic resource bundles confirmed SHA-1/SHA-256 updates,
preservation of a separate logo entry and rejection of an oversized tree.
No MTD/UBI operations, boot-chain build or hardware execution occurred. The checked
boot-builder source hash was
`e0fc847bd20a621f85190ea5fd15b1f4a8ce36caec960823607e32e81583e5d3`;
pairing importer hash
`24bd520c8f5d9a2767acef52dcc8d0c4c790fa7fa24f3f9c4d8e321ad2669065`.
