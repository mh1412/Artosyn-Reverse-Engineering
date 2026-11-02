# AR9311 P1 camera, ISP and encoder

[SoC studies](README.md) · [P1 Air](../devices/betafpv-p1-air.md)

Scope: the SC231AI-equipped P1 Air, source Linux 6.18.36, and the camera work
validated on 2026-11-01 UTC. Related
[NT99235 reference work](ar9311-nt99235-camera.md) supplied implementation ideas;
its sensor configuration and image-quality conclusions were not P1 validation.

## Data path and ownership

```text
SC231AI RAW10 → CSI → VIF → ISP → CVISP YUV420M
                                      │
                               exported DMA-BUFs
                                      ↓
                                Wave521C HEVC
                                      ↓
                            recording / Kestrel packets
```

The sensor, CSI, VIF, ISP, CVISP, host encoder driver and automatic controllers
are built from source. ISP tuning and codec microcode remain extracted binary
inputs. No vendor media application or runtime library is required by this path.

CVISP capture supplies three-plane YUV420M buffers. The encoder imports them
through DMA-BUF, and capture regains ownership only after the encoder returns
its OUTPUT buffer. Only compressed HEVC bytes are copied into the application's
packet ring. “Zero CPU pixel copies” does not mean no compressed-byte copies or
no hardware memory traffic.

The initial source mode is complete-frame 1920×1080 HEVC at 60 fps. The recorder
requests 8 Mbit/s and a 60-frame GOP. Those are requested encoder parameters;
measured encoded size varies with scene and control behavior.

## P1 memory layout

The source DT isolates each DMA owner within the board's 256 MiB memory span:

| Start | Size | Owner/reservation |
| --- | --- | --- |
| `0x20000000` | 128 MiB | Linux region, including reservations |
| `0x27400000` | 5 MiB | Preserved stock QSPI reservation inside the Linux region |
| `0x28000000` | 40 MiB | CVISP capture and owned scratch |
| `0x2a800000` | 8 MiB | ISP |
| `0x2b000000` | 8 MiB | VIF |
| `0x2b800000` | 71 MiB | Codec |
| `0x2ff00000` | 1 MiB | Early RAM-loader/secondary-CPU reservation |

The original 32 MiB CVISP pool was too small once the queue requested five
buffers plus scratch. The final allocation increased CVISP to 40 MiB and
reduced the ISP pool to 8 MiB. Driver-owned scratch and translated ISP DMA
descriptors prevent idle capture from writing into released application buffers.

## Sensor and ISP initialization

The fixed SC231AI mode uses RAW10, two CSI lanes, a 24 MHz sensor clock and
378 MHz link frequency. Measured CSI IPI timing is HSA/HBP/HSD `16/16/33`.
P1-specific RAW10 geometry and ISP mode `0x58870780` were matched against the
stock setup.

Initial white or dark/noisy output required corrections to black-level control,
DRC length, Bayer interpretation, gain-dependent tuning and white balance.
Frame interrupts and encoder output syntax were insufficient acceptance tests:
some earlier runs produced corrupt or black pictures despite plausible counters.

## Restart defect

A full camera-module reload originally produced black frames while frame
interrupts continued. Register-isolation experiments reproduced the failure by
clearing the VIF AXI enable/run bits at offset `0x190`. The vendor shutdown path
leaves those bits set.

Preserving the AXI state during stop and initialization fixed the tested reload
path. Clean-build validation then recorded 6,000 frames at 59.969 fps and,
after a full module reload, 600 frames at 59.598 fps with visible content.

This fix does not explain every earlier anomaly. One historical cold boot ran
at approximately 17–23 fps; later cold boots passed, but the cause of that
isolated slowdown was not established. Capture counters also recorded startup/
idle drops, so correct complete recordings are not a zero-drop claim for every
stage of the pipeline.

## Automatic exposure

The controller adapts a 478-entry P1 exposure table to grouped SC231AI exposure,
analogue-gain and digital-gain controls, and updates gain-dependent ISP tuning.
The sensor gain encoding was checked against 348 extracted gain steps.
Controllers start after eight warm-up frames.

An initial zero-luma divide/log-overflow defect was fixed and covered by host
regressions. Lit-scene tests demonstrated convergence; they do not establish
identical behavior to the vendor algorithm under all motion or illumination.

## Automatic white balance

P1's ISP statistics order is **GRBG**: green channels 0 and 3, red 1 and blue 2.
This is distinct from the **BGGR sensor bus** order. The measured statistics
layout is 16 horizontal zones by 36 vertical blocks, with 826 samples per
channel per tile. Correlation with the decoded scene supported the channel
mapping; small geometry offsets remained provisional.

The source controller:

- Reads fresh, coherent statistics at 10 Hz.
- Rejects dark, clipped, malformed, strongly colored and green-disagreement zones.
- Requires at least 32 usable zones across eight blocks.
- Uses median channel ratios and fixed initial board gains as its selection anchor.
- Bounds red/blue gains to 128–1024 in Q8 and limits changes to eight units per update.
- Holds gains when neutral evidence is sparse and fails after roughly five seconds
  without fresh coherent statistics.

In the lit bench scene, gains moved from red/blue `313/807` toward `355/659`.
A 1,200-frame run at 59.531 fps decoded fully. Ratios in two visually neutral
regions moved closer to one. These were visual candidates, not a calibrated
color chart. Mixed illumination and scenes dominated by one color remain
limitations; fixed gains are available.

## Firmware identity

| Input | Bytes | SHA-256 |
| --- | ---: | --- |
| P1 `chagall.bin` | 999,616 | `308b96351168cc7dbbe936de1490f9ab553aa9ae7e1679d8e9a8c4ab8cd5bf86` |
| P1 `chagall_lowmem.bin` | 999,360 | `c70476d90549440408c846b70447035978771ef4e24a1f0003677e2d939ca49b` |
| P1 1.0.44 SC231 tuning | — | `8301d0d41cd5c413730e94315a3edeb2036789828937e061ac62748f0a31de23` |
| Ascent reference `chagall.bin` | 998,144 | `99bc74cdf1e7ded785c2b65440c983f4e128727eb2bbdf309b14ddde668fd8ac` |

The source P1 encoder uses the low-memory P1 firmware. The Ascent blob differs;
matching the Wave521C product ID alone is not grounds to substitute it.

## Integration limits

Kestrel owns encoder buffers, packetization and AE/AWB process lifetime. Its
AR9311 backend excludes HiSilicon-specific MMIO and board commands. Bitrate
changes and keyframe requests use V4L2 controls.

Slice output, intra refresh, alternate resolutions, rotation, remote ISP tuning,
FC UART and P1 temperature/battery telemetry were outside the implemented
backend. Successful complete-frame streaming does not establish those features.
