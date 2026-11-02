# Gnd485 hosted-browser installation and restoration

[Experiments](README.md) · [fpvOS update design](../firmware/gnd485-fpvos-update.md) ·
[Browser interfaces](../systems/ascent-browser-interfaces.md)

On October 31, 2026, the hosted fpvOS flasher installed the exact
`2026.10.31-ascent-vrx.1` release on a physical standard Ascent VRX, updated it
again, verified it after a power cycle and restored the original stock boot
images. This later run supersedes the release-publication note that the exact
build had not yet been hardware flashed.

## Tested identities

| Item | Recorded identity |
| --- | --- |
| Board | `FPV-Ascent-Gnd-485-V1.2-1.0`, factory serial `2614` |
| Initial and restored firmware | `Ascent_G_Gnd_18_21_10`, SDK `18.21.10` |
| Initial USB identity | `1d75:0101`, USB serial `CX11` |
| Stock root selection | `userapp0`; fpvOS used `userapp1` |
| fpvOS source commit | `8cb13d6eed4e5b913625f79f675dfd957825cb18` |
| Tested website deployment commit | `5cdb138ccd66c65d0ce0c63dae4346883bae96a1` |
| Browser | Chrome `156.0.8078.4` on Linux x86-64 |
| Installed kernel | `6.18.36-fpvos-ar9311` |

The browser loaded the hosted application over HTTPS. The retained harness
drove its ordinary controls through Chrome DevTools and verified native
`navigator.serial`; it did not replace Web Serial with a simulated receiver.
Initial stock-to-updater port selection used the actual browser chooser in an
isolated Xvfb display. Other stages used headless Chrome with native serial
access. The harness also checked the physical USB topology and required a
single matching updater before choosing its port.

The updater's USB serial `fpvos-ar9311` is a shared backend identifier, not a
unique factory serial. Its reported SDK `development` and temperature `0 °C`
are likewise not measurements of the stock SDK or board temperature. Matching
the selected physical receiver matters when its USB identity changes.

## Artifacts and integrity

| Artifact | Bytes | SHA-256 |
| --- | ---: | --- |
| Full install/self-update bundle | 10,796,694 | `fc76aa1fe7c702176063c15b7eee87f88b9df2f2f0eb0fd4aa45513bda108ff4` |
| Original kernel/DTB restoration bundle | 6,685,184 | `2f2f9c1917c39304cbd20a4b0b6250a7037fb2351fc22c1bf9a24abd4722cfd3` |
| Bootstrap manifest | — | `036631de06a7e1d7d69b494519ad9fbb2fe85ff82d18d990501aa48d0ed2ba04` |

The manifest pins a 1,418,832-byte static updater and a 433-byte launcher.
Its `v2` field describes the bootstrap schema; the release directory and pinned
hash identify the October 31 assets. Publishing the firmware release did not
itself change the website's asset pin. The later deployment above did.

During documentation review, the retained install and restoration files were
hashed again and matched the browser audits' lengths, MD5 and SHA-256. The
bootstrap manifest and both asset hashes also matched. These are offline
artifact checks, not a new hardware run. The bundles are unsigned: integrity
hashes do not establish publisher authentication.

## Recorded sequence and results

| Stage | Observation |
| --- | --- |
| Initial connection | Native Web Serial read the expected stock board, firmware and factory serial |
| Stock-to-fpvOS installation | UI reported 48 seconds; reconnected device reported the exact install-bundle SHA-256 |
| fpvOS self-update | Same bundle installed through clean RAM recovery; UI reported 58 seconds and the same postboot SHA-256 |
| Power cycle | PSU output was disabled and measured 0 V/0 A, then enabled; subsequent browser connection reported the same release SHA-256 |
| Stock restoration | Full original kernel/DTB bundle installed; UI reported 37 seconds and restored firmware, hardware and factory serial matched the initial identity |

The browser audits ran between 18:23 and 18:28 UTC. The durations are the UI's
reported operation times, not independently measured NAND programming times
or repeated performance benchmarks. All five audits completed without recorded
browser errors. The simulated harness regression retained alongside them is
separate evidence; its shorter fixture timings are not physical measurements.

Serial preflight identified stock bank 0 and fpvOS bank 1 before installation
and in later recovery. Earlier bootstrap-v2 work reported both stock bank
selections, but this exact-release run does not add a bank-1 stock test.

The PSU settings remained 12 V with a 1.5 A current limit. After restoration,
the final reading was 11.99 V, 0.193 A, with no reported protection trip. The
immediate power-on sample was transitional and is not the settled supply voltage.

## Platform checks and evidence limits

The installed platform check reported zero failures. It observed both CPUs,
a RAM root with no NAND-backed filesystem mounted, UART and watchdog devices,
crash storage, configured USB ECM, both MMC clocks at 100 MHz, the radio node
with zero bus errors, Wave5 decoder registration and DRM registration.

That check explicitly left RTC ticking/readback unqualified, noted the lack of
verified watchdog reset control, and did not test host ECM reachability. SD
writes and memory tests were skipped. Registered devices do not prove RF
reception, decoded pixels or an HDMI signal. The
[separate combined platform regression](ar9311-platform-validation.md#later-installed-platform-regression)
contains earlier media/storage observations; they must retain their own build
and test scope.

The attempted shell read of `/etc/fpvos-release` found no file. Release identity
here comes from the postboot serial protocol's whole-bundle SHA-256, corroborated
by the retained artifact, rather than that filename. The initial stock shell
capture also failed to execute its intended command, so it supplies no new
pre-install partition-hash measurement.

The run establishes a browser-mediated install/update/boot/restore round trip
for one named receiver and stock version. It does not establish interrupted-write
recovery, physical bad-block handling, other board revisions, every advertised
stock version, RF/video operation on this exact build, or whole-device byte
identity after stock software resumes. Shared kernel/DTB writes remain
non-atomic across power loss.
