# VRX Pro fpvOS application and video pipeline

[Systems](README.md) · [Open receiver comparison](open-receiver-stacks.md) ·
[Boot and kernel](../firmware/vrx-pro-fpvos-boot.md)

The retained fpvOS 2026.10.beta4 documentation describes an RK3568 Ascent VRX Pro
receiver with Kestrel, an open radio host stack, source-built Rockchip kernel/MPP
and Panfrost graphics. This is distinct from the AR9311 Gnd485 port and its
stock-kernel software UI. The application behavior below is reported by that
snapshot; the wiki review did not operate a VRX Pro or replay its flight captures.

## Components and ownership

| Layer | Examined design |
| --- | --- |
| Image and startup | Buildroot fpvOS image, adapted stock boot chain, SD root |
| Application | Kestrel ground process: radio control, video, HUD, menus, recording and browser view |
| Host radio | ar_libre `arlink.ko` plus ABI-compatible client library, replacing the vendor daemon |
| Radio processor | Open8030 by default in this snapshot, with stock firmware selectable |
| Decode | Source-built Rockchip MPP, with FFmpeg fallback in the application |
| Display | DRM/GBM, separate video and GLES HUD planes; Mesa/Panfrost |
| Screen composition for recording | Vendor `librga` combines decoded picture and HUD into encoder input |

Older prose calling the GPU and all media libraries proprietary predates this
selected graphics/decode path. The [packaging study](../firmware/vrx-pro-fpvos-boot.md#packaging-and-checks-performed-here)
checks the source/vendor exclusions. Rockchip MPP is separate from Artosyn MPI,
and HiSilicon's air-side camera/encoder libraries remain prebuilt inputs.

The examined Open8030 package pins the gehee branch at
`382904174fb788ea3c69ed10c8df8590b8e6e7ab`. Its published profile archive is
version `20261022`, SHA-256
`4c7bbab5a9945f6480083381b2a424750a6d892d928577d25d4b6c474081fe1d`;
the recipe checks the archive and its per-file checksums. VRX Pro selects its
USB profile and `AR_FORCE_USB`; Lite/Lite+ selects an SDIO AP profile with
optional factory data. The Gnd485 profile remains a separate private input.
Role, transport and profile are explicit build selections, not inferred from
the host application's name.

This package identity is not the optical test's `ed20c01` identity or the earlier
mh1412 v0.2.0 SDIO release. Published Pro/Lite profiles do not retroactively make
every earlier private-profile experiment reproducible from that archive.

## Slice decode and presentation

The retained rkvdec2/MPP notes describe starting a picture on its first slice,
appending later parts and reporting when the top rows are ready. The kernel
interface includes `MPP_CMD_STREAM_APPEND` and `MPP_STREAM_WAIT`; MPP exposes
stream-start/append/top-readiness controls. Separate pages for appended chunks
avoid hardware reads crossing an append boundary. A damaged PPS referencing a
missing SPS is rejected before decoder submission.

Stream mode disables the decoder's linked-task mode at probe, leaving one task
at a time. The documented defaults allow 400 ms waiting for another part within
the task's 500-ms lifetime, plus a separate 50-ms active-work watchdog. These
are timeout controls, not normal latency measurements. Runtime switches can
disable application streaming, early presentation or split planes; changing
the kernel's probe-time linked-task selection requires a new boot configuration.

| Presentation mode | Reported behavior |
| --- | --- |
| SPLIT | Present the top before the bottom is ready; may show a temporal seam; disabled on the described 60-Hz path |
| FULL FRAME | Wait for the whole picture's arrival; default |
| FULL DECODE | Wait for complete decoding before presentation |
| Refresh Sync ON | Synchronize where display refresh is a whole multiple of camera rate |
| Refresh Sync FREE | Stretch refresh timing toward the camera rate when the display can run faster |

FREE does not make a 120-Hz screen show every 144-fps camera picture. Timing to
the last slice with `display_lock_slice: 2` is distinct from presenting the top
half early. Sync margin adds 0–4 ms to presentation timing. Display-mode and
refresh changes have a fifteen-second confirmation/revert window in this
snapshot. See the [reported optical measurements](../experiments/vrx-pro-latency.md)
for their precise scope.

These mechanisms cannot be transferred to [AR9311 slice decoding](../experiments/ar9311-slice-decoding.md)
by analogy: forcing an incomplete Wave521C picture to finish did not establish
that the same picture could resume when later slices arrived.

## Video, recording and applications

The interface distinguishes radio search, link-up, application handshake and
waiting for a decodable picture. A radio connection therefore does not establish
fresh video. The guide's red warning includes no frame for 400 ms; its dimmed
idle state follows roughly 1.5 seconds of quiet link activity.

FPV recording stores the air stream without the HUD, with a repair path for
missing pictures. SCREEN recording uses a separately composed picture plus HUD,
encoded as H.265. The guide describes up to 100 fps, including every other
refresh on a 120-Hz display; this is not a claim of recording every 120-Hz frame.
The gallery supports playback, seeking and deletion. Recordings use an exFAT
partition mounted at `/media/dvr`; before a new recording, the application can
delete oldest `fpvOS_` recordings when less than 300 MB remains. It does not
automatically start recording on arming in this snapshot.

The optional Wi-Fi spectator path re-encodes the composed view as H.264, up to
30 fps within 5 Mbit/s. It is another encode/network/browser path, not the local
HDMI latency path. The AP is off each boot, supports up to four clients in the
guide, and also exposes recordings. SSH is configured on the USB network rather
than that Wi-Fi interface. Settings and bindings persist on the card; channel
selection is session-scoped while requested power is saved per air unit. UI power
labels do not constitute emitted-power measurements.

Debug mode retains up to thirty receiver logs beside recordings and up to five
armed-flight logs on the air unit. Ordinary runtime logs disappear at power-off.
Useful evidence records both application and radio versions, the air camera,
recording mode and stage at which video stopped. Pairing data is per-device
configuration, not a generic firmware artifact.

## Compatibility and known failures in this snapshot

The guide's broad statement that a stock air unit links needs its troubleshooting
qualification: Open8030 on the Pro receiver with stock radio firmware on the air
unit was reported to produce missing or damaged picture portions, especially
around rate changes. The proposed controls used matching radio families at both
ends. This is a version-specific report, not proof that all native/stock pairs
fail; the separate [Ascent SDIO receiver experiments](../experiments/open8030-early-dev.md)
have their own successful original-transmitter observations.

One hot-plug incident reportedly delivered video portions twice and persisted
through channel, screen-mode and application changes until receiver power cycle.
Its cause was unresolved. Small flight artifacts were also reported. These
limits belong beside the feature description rather than being erased by a
successful latency run.

The Lite+ OS02K10 fast modes were 864p120/144, with 1080p limited to 100 fps in
the guide. Lite's CV2004 camera and IMU differ. Air-image bitrate controls and
capabilities do not describe the stock encoder's policy, nor the separate P1
SC231AI source pipeline. The [air application study](kestrel-air.md) describes
that platform split and timing metadata in more detail.
