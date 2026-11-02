# Ascent stock compressed-video extraction

[Systems](README.md) · [Ascent VRX](../devices/ascent-vrx.md) ·
[Backend control](../protocols/ascent-backend-control.md)

A passive loopback capture on Gnd485 17.5.3 recovered stock compressed HEVC
before decoding. This study records that stock stream and candidate extraction
boundaries. It does not establish optical latency or a stock network-video server.

## Data path and ownership

```text
VTX encoder → RF → daemon_sdio TCP 50000 → ar_ldy_gnd → decoder/display
                       │
                 passive capture → protocol reassembly → HEVC inspection
```

The captured stream uses socket-data command `0x04020003`. This is a
message command, not TCP port 3. Its 18-byte header starts with `aa`, contains
a little-endian payload length, big-endian command and complemented-XOR header
checksum, and ends after the payload with `bb`: 19 bytes total overhead.
The payload mixes Annex-B HEVC and vendor metadata; blocks beginning with the
forbidden bit set are not ordinary HEVC NAL units. Header validation and removal
of metadata precede decoding. See also the
[host-link format](../protocols/ar8030-host-link.md) and
[application/video framing](../protocols/ascent-application-streams.md).

## Passive extraction, October 5

The initial receive-only AF_PACKET capture copied the existing loopback flow;
it did not consume application queues. Bounded capture files excluded SSH and
duplicate outgoing loopback packets. Temporary receiver files were removed and
the original daemon/backend/UI processes remained running.

| Stage | Evidence and result |
| --- | --- |
| Steady-state capture | 1,729 packets, 6,517,222-byte pcap; no reported drops |
| Reassembled TCP | 6,366,527 bytes without gaps, overlaps or conflicting segments |
| Initial HEVC extraction | 6,291,293 bytes of TRAIL_R slices; no VPS/SPS/PPS or IDR, so independent decode failed |
| First startup trigger | Did not fire; 24-byte empty pcap, no playback result |
| Capture after resumed video | 2,519 packets over 6.004977 seconds; 1,808,297-byte pcap, no reported drops |
| Reassembled startup flow | 1,583,306 bytes without gaps or conflicts |
| Extracted HEVC | 1,487,806 bytes with VPS/SPS/PPS, SEI and IDR; 316 decoded 1920×1080 Main-profile frames |

A read-only check found the stock `g_video_param_buf` cache length zero, so it
could not supply the missing bootstrap headers. The operator restarted the VTX
twice during capture development. This establishes why observing P slices alone
is insufficient for independent playback.

Vendor metadata reported 60 fps. Stream-copy remux produced 316 frames over
5.266667 seconds without re-encoding. The raw elementary stream's default 25 fps
probe result was not source timing; neither remux duration nor the approximately
12.7 Mbit/s steady-state flow measures glass-to-glass latency. The final
unterminated NAL was omitted.

## Alternative export boundaries investigated

The original feasibility study found RTSP symbols and a `/live.ch01` route hint,
but no listener on TCP 554. The 17.5.8 reference backend, SHA256
`5c07452b91a5284b029c03b92636bb6172e04aa2a27a3b62fb832e8729495a0a`,
contains `main_rtsp → test_mini_rtsp`, registering that route and starting
ports 554/8060. Its route-open callback calls `fpv_record_stop` before opening
the stream. This is an offline lead with a recording side effect, not a verified
passive export endpoint in the running 17.5.3 image.

A second candidate was the existing decoder submission boundary. In the exact
live 17.5.3 backend, `uav_ar_vdec_send` at `0x43fd38` builds a stream descriptor
under a decoder mutex and calls `AR_MPI_VDEC_SendStream` through PLT entry
`0x40e260` at callsite `0x43fe24`. A version-matched hook could copy compressed
data before stock buffer reuse, but that hook was not validated in this stock study.
Any such design needs bounded copying/ownership and an independent sending
worker; a blocking network operation inside that mutex would affect stock decode.

The initial collector found FFmpeg libraries but no `ffmpeg`, `ffprobe`,
GStreamer launcher, tcpdump or strace executable through its PATH lookup. It
also found no separate baseband-client library in the inspected locations:
the stock backend embeds client code. Neither library names nor missing PATH
entries establish a ready-made streaming service or a complete filesystem search.

Compressed export also avoids a basic bandwidth mismatch. At 1920×1080×60,
uncompressed 8-bit YUV420 is about 186.6 MB/s / 1.49 Gbit/s; four-byte RGB is
497.7 MB/s / 3.98 Gbit/s. Both exceed nominal USB 2.0 bus rate before overhead.
These are format arithmetic, not measured receiver throughput. The exported
compressed stream excludes stock UI/HUD composition added later by the display.
