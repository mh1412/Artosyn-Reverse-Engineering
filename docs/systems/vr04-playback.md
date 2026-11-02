# VR04 playback, clip metadata and recovery

[Systems](README.md) · [Open media overview](vr04-open-media.md)

## Scope

This study follows MissingLynk revision
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13`: the receiver process, interactive
playback implementation, HUD player and recording-list reader. The command
dispatch was checked separately. These are source findings; no new decoder,
display or recovery test was performed. Reported hardware results remain in the
[media validation record](../experiments/vr04-media-validation.md).

## Two file-playback entry points

Interactive playback runs inside the RF process and temporarily replaces its
live decode graph. The standalone file argument enters a different path.

| Behavior | Interactive player | Standalone file argument |
| --- | --- | --- |
| Container | Case-insensitive `.mp4`/`.mov` selects `qtdemux`; other files enter an elementary-stream parser | No demuxer; expects an elementary stream |
| Codec selection | MP4 probe selects H.264/HEVC; raw `.h265`/`.hevc` suffix selects HEVC, otherwise H.264 | Case-sensitive `265` or `hevc` anywhere in the pathname selects HEVC |
| Allocation metadata | Advertises `GstVideoMeta` to preserve the decoder's DMA-buffer layout | Does not install that allocation-query probe |
| Scaling | Can use an existing composite pool and scaler | Uses single-stream scanout |
| End of stream | Holds the last frame and stays in playback mode | Quits the main loop and tears down display/decoder |

Both sinks use timestamp synchronization, a two-buffer queue and disabled
appsink dropping. The file callback can still discard a frame when mapping,
DMA-buffer import, framebuffer creation, pool acquisition or scaling fails.
Its normal flow return does not count those drops as a fatal pipeline error.
The optional plane-ID argument is parsed but unused in both top-level modes.

## Codec and geometry detection

The interactive MP4 probe walks top-level atoms to the first `moov`, handling
ordinary and extended sizes. It reads at most 8 MiB of that body, then searches
byte by byte for the first `avc1`, `hvc1` or `hev1` sequence. Width and height
come from offsets 28 and 30 after that sequence. It does not validate the
sample-entry hierarchy or select a particular video track. A matching sequence
elsewhere in metadata can therefore select the codec or geometry incorrectly.

Only 1920×1080 is classified as native. An unreadable or unrecognized probe
defaults to H.264 and nonnative geometry; this is not a guarantee that an
unparsed HEVC file will play. Raw-file codec selection uses the suffix, but the
geometry decision still comes from this MP4 probe.

Scaling requires both an open `/dev/arscaler` and an already allocated composite
pool. Playback does not allocate that pool itself. If either is unavailable,
the code selects single-stream scanout even for a nonnative clip. Consequently,
the presence of the scaler alone does not establish working sub-panel playback
after starting in direct tile mode. In the scaled path, three I420 plane
operations fill a 1920×1080 composite; in the native path, the display item
retains the decoder sample until retirement.

## Recording list and duration are separate checks

The list admits case-insensitive `.mp4` names whose `stat` size is at least
64 KiB. This is a size heuristic, not a decode test: it can hide a small valid
clip or retain a larger unusable file. It does not explicitly require a regular
file. Entries are collected up to the caller's limit and then sorted by name
descending; that does not guarantee the newest entries across a larger directory.

The HUD's duration reader walks file-bounded top-level atoms and reads `mvhd`,
falling back to `mvex/mehd` when the movie duration is zero. It needs a nonzero
movie timescale and rejects a computed length above 24 hours. Missing metadata
does not cause a fragment-by-fragment duration reconstruction. Top-level
extended sizes are supported, but size-zero atoms stop the walk; nested walkers
handle ordinary sizes. These limits matter when importing files from other
recorders.

A 64-entry duration cache uses basename, byte size and second-resolution
modification time. It also caches failed reads as zero. A replacement preserving
those values can retain a stale duration. When full, the cache replaces slot
zero rather than implementing a general eviction policy.

The HUD prefers the header-derived duration for its progress bar. Separately,
the pipeline polls position and duration nominally every 200 ms and retains the
largest positive duration seen. That prevents a decrease; it **does not prevent
the reported duration from growing**. The HUD falls back to that value when it
has no metadata length. These two readers must not be described as one fixed
duration calculation.

## End of clip, replay and control state

At EOS, the interactive player removes its position timer, requests PAUSED,
sets the paused and ended flags, and pins position to the last known duration
when available. It keeps the graph and display buffers. The HUD's center key
then sends a fresh PLAY for the same clip, rebuilding the graph and resetting
the ended state and timer.

Direct RESUME, SEEK and SPEED commands do not perform that EOS reset. In
particular, speed selection clears paused but leaves ended set and does not
restart the position timer. The HUD's left/right speed keys have no ended-state
guard. This is distinct from its center-key replay behavior.

Speed zero is normalized to 1×. Rate changes use a flush/key-unit seek, adding
trickmode flags away from 1×; negative rates seek over the interval from zero to
the queried position. Position seek requires a positive queried duration and
clamps its argument to 1000 permille. Seek and state-change return values are
not checked before publishing the resulting software flags. A speed label or
unpaused flag therefore describes requested state, not accepted decoder behavior.
The reported forward-speed tests do not qualify reverse or every flush/restart
case.

## Teardown and return to live

Entering playback shuts down display before stopping RF decoding. A stream-only
RTSP encoder is stopped to release codec resources; an active recording is left
untouched by this transition, although live composites cease. Returning to RF
can restart the stream-only encoder. This policy does not establish uninterrupted
recording across playback.

Stop removes the playback timer and bus watch, shuts down scanout, requests
decoder NULL and waits with `GST_CLOCK_TIME_NONE` before releasing the graph.
RF teardown also joins the receive thread and waits indefinitely for decoder
state. These calls impose no overall recovery deadline.

Fatal playback errors request live recovery. Display reinitialization failures
are logged, and the recovery helper ignores the RF-start return value. RF start
itself does not check the PLAYING transition or thread-creation result. Likewise,
interactive playback can publish active state after a display-init failure or
unchecked PLAYING request. “Return to live” describes an attempted transition,
not a guarantee of fresh RF pictures. A failed control-socket bind can also leave
the receiver running without external playback controls.

## What the HUD's loading indicator proves

The pipeline sets its rendering flag after submitting a display item. This
does not wait for a display latch or optical observation. The HUD reveals its
transport controls when that flag arrives; otherwise it cancels loading on EOS
or after a nominal eight-second UI deadline and sends STOP. That deadline bounds
the HUD's wait, not the pipeline's potentially blocking teardown.

Decoder-output frame counters and the standalone FPS summary measure decoded
buffers. They do not establish panel refresh or visible motion. The retained
black-panel episode despite healthy software counters remains relevant; see
the [visible-output limits](vr04-open-media.md#playback-and-visible-output-limits)
and [receiver latency study](../experiments/vr04-receiver-latency.md).
