# VRX Pro beta4 reported optical latency

[Experiments](README.md) · [VRX Pro pipeline](../systems/vrx-pro-fpvos.md) ·
[Latency measurement boundaries](../systems/video-latency-comparison.md)

The retained fpvOS 2026.10.beta4 latency report contains an LED-to-photodiode
bench measurement for an Ascent Lite+ and VRX Pro. These are the report's
results, not new measurements performed during this wiki review. Its raw optical
samples and instrumentation were not independently reprocessed here.

## Test identity and method

| Component | Reported setup |
| --- | --- |
| Air | Ascent Lite+, OS02K10, beta4 air image, Open8030 |
| Receiver | Ascent VRX Pro, beta4, Open8030 `ed20c01` |
| Display | 1920×1080 HDMI headset panel, 60/120-Hz modes, no FreeSync |
| Radio | 5,914 MHz, 20-MHz width, MCS11, short frame, bench |
| Instrument | ESP32-S3-controlled LED in front of camera and photodiode on screen |
| Sampling | About 100 readings over 75 seconds per setting, after application restart and twenty-second settling |

The optical interval begins when the LED turns on and ends when the measured
screen region brightens. It includes the camera's phase wait and panel response.
The report estimates median variation between runs at roughly ±1–2 ms. Small
differences at that scale are not strong evidence of a faster configuration.

Its internal **capture → lit** metric uses shared-clock camera/display timing
and excludes camera phase wait and panel delay. The user guide also labels a
software stats field “glass to glass”; that label does not make software timing
equivalent to the external photodiode measurement. With stock air software, the
guide explicitly estimates parts of the air-side timing.

## Selected optical results

Values are medians in milliseconds, with 10th–90th percentile intervals.
Main comparisons used zero sync margin unless stated otherwise.

| Camera / bitrate | Presentation | Optical median (p10–p90) |
| --- | --- | --- |
| 864p120 / 12 Mbit/s | SPLIT, FREE | 24.6 (21.9–28.0) |
| 864p120 / 12 Mbit/s | SPLIT, FREE, timed to whole picture | 27.5 (25.2–30.5) |
| 864p120 / 12 Mbit/s | FULL FRAME, FREE | 29.1 (25.7–31.0) |
| 864p120 / 12 Mbit/s | FULL FRAME, sync OFF, default | 29.8 (26.0–34.5) |
| 720p100 / 12 Mbit/s | SPLIT, FREE | 25.9 (22.4–29.6) |
| 720p100 / 12 Mbit/s | FULL FRAME, FREE | 29.6 (25.1–33.5) |
| 720p100 / 12 Mbit/s | FULL FRAME, sync OFF | 31.3 (26.1–36.1) |
| 1080p100 / 20 Mbit/s | SPLIT, FREE | 30.0 (25.9–33.5) |
| 1080p100 / 20 Mbit/s | FULL FRAME, FREE | 35.2 (31.3–38.4) |
| 1080p100 / 20 Mbit/s | FULL FRAME, sync OFF | 36.8 (31.6–42.1) |

The lowest SPLIT numbers concern the top of the picture. The report places the
bottom roughly 8–10 ms later; a temporal seam is part of that tradeoff. At
864p120, its internal top/bottom timings were 16.0/24.4 ms. Whole-picture timing
used `display_lock_slice: 2`, aligning presentation to the last slice instead
of showing the top early.

“Smoothness” was defined as the fraction of pictures shown one camera period
after the preceding one: at least 95% was called very smooth, 90–95% smooth,
and 85–90% mostly smooth. These are the report's categories, not standardized
quality grades. FREE measured about 119.98 Hz for 120-fps input and 100.15 Hz
for 100-fps input. ON did not remove the 100-to-120-Hz mismatch on this panel.

At 864p144 on the 120-Hz panel, roughly one picture in six could not be shown.
The detailed settings ranged from 28.8 to 30.0 ms, within the reported variation,
and all juddered. A summary table's 28.8-ms “lowest” entry belongs to whole-picture
timing in the detailed table; ordinary SPLIT/FREE was 28.9 ms. The distinction
does not establish a meaningful latency advantage at that noise level.

## Sync margin and limits

Separate single-run 720p100/FREE comparisons changed margin from 0 to 2 ms:

| Mode | Optical medians, 0 → 2 ms margin | Pictures a refresh late, 0 → 2 ms |
| --- | --- | --- |
| Whole-picture timing | 29.8 → 30.3 ms | 6.6% → 1.0% |
| Early SPLIT | 24.9 → 27.6 ms | 11.5% → 9.6% |

Internal capture-to-display timing moved by the full two milliseconds. The
whole-picture optical median moved only 0.5 ms, within the stated run variation;
the measured improvement was primarily smoother delivery. The summary's
30.3-ms smooth 720p100 result includes this margin and must not be confused
with the main zero-margin table.

This is not an optical comparison against DJI, stock Ascent, Gnd485, P1 or a
different screen. Lite uses a different sensor. The report does not establish
range, interference tolerance, long-duration freeze rate or worst-case latency
outside the sampled windows. Its successful slice-stream path also does not
resolve the separate AR9311 resumable-decoding investigation.
