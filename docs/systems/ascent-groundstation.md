# Ascent compressed-video ground station

[Systems](README.md) · [Ascent VRX](../devices/ascent-vrx.md) ·
[Backend control](../protocols/ascent-backend-control.md)

On the Gnd485 receiver running the studied 17.5.3 stack, compressed HEVC was
first extracted passively and then exported live through a transparent radio
TCP proxy. Persistent startup survived full reboots; subsequent recovery evidence
confirmed receiver decoding and direct 1080p host playback. Physical HDMI output
through the proxy and end-to-end latency were not independently measured.

## Data path and ownership

```text
VTX encoder → RF → daemon_sdio TCP 50000
                           ↕ one upstream connection per backend connection
                    transparent proxy TCP 50001
                           ↕ scoped connect redirect
                      stock ar_ldy_gnd → stock decoder/display path

proxy's copied incoming video → HEVC extraction → RTP/UDP → host HEVC decoder
```

The proxy relays the **entire vendor protocol in both directions**. It does not
open a second radio receive socket or replace stock media ownership. Eight
backend sessions were observed; defaults allow 16 clients, with a maximum of 32.
Each connection uses bounded 64 KiB buffers per direction, nonblocking I/O and
TCP half-close propagation.

The extraction branch selects socket-data command `0x04020003`. This is a
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
data before stock buffer reuse, but that hook was not the deployed proxy.
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

## RTP export and bounded loss behavior

Only copied video enters a preallocated queue of 128 × 8 KiB slots by default,
127 usable. The forwarding thread does not wait for the video worker or host.
Queue overflow drops copied bytes; offset gaps discard partial parsing/pictures
and return extraction to keyframe wait. Nonblocking UDP send failure likewise
returns the video branch to keyframe wait. **The proxy process itself remains
required for the stock TCP path.** Isolation of video-copy congestion does not
provide failover if that process dies.

The worker caches VPS/SPS/PPS, uses HEVC first-slice flags for picture boundaries,
and obtains timing from metadata, allowing absent/interleaved metadata. RTP uses
payload type 96, a 90 kHz clock, H.265 single-NAL/fragmentation packets and one
marker per complete access unit. Default payload is 1,200 bytes, configurable
from 64 to 1,400. SDP is written atomically after all codec parameters arrive.

Startup requires parameters and IDR/BLA, NAL types 16–20. There is no cached-GOP
replay, CRA bootstrap or implemented keyframe request. Stream ownership and codec
parameters reset on reconnect or detected clock/frame discontinuity. Payload/NAL
limits are 1 MiB, access units 4 MiB and parameter sets 4 KiB each. Malformed or
oversized video causes extraction resynchronization without changing relay bytes.

Recorded host replay forwarded 1,583,306 bytes unchanged and produced 1,916 RTP
packets and **315 complete decoded frames**, without reported queue/UDP loss or
decoder errors. This differs from the passive 316-frame result because the
proxy deliberately withheld the final unfinished picture. It is a replay
validation, not a receiver capacity benchmark.

## Redirect and persistent boot

The backend embeds `127.0.0.1:50000`. A preload shim intercepts `connect` only
for executable basename `ar_ldy_gnd`, IPv4 `SOCK_STREAM`, and that exact
destination. It changes a local address copy to port 50001, preserving normal
nonblocking/EINPROGRESS behavior. Other processes, addresses, ports and UDP
pass through. `GND_RADIO_PORT=50000` explicitly bypasses redirection; invalid
configuration yields EINVAL for matching connections.

The executable is statically linked; the shared shim was built against the
receiver's glibc 2.25 ABI, with only GLIBC_2.17 version requirements and no
RPATH/RUNPATH. This ABI inspection alone was not the execution proof.

An attempted backend-only replacement failed at `AR_MPI_VB_SetConf`, including
when retried without the shim. Kernel media-buffer state survived process
termination; the attempted `AR_MPI_VB_Exit` path returned EINVAL. Cold startup
was required. The first successful installation used a one-shot hook; the
October 6 installation subsequently made the proxy persistent.

Persistent artifacts reside on receiver UBIFS under
`/usrdata/ascent-groundstation/`. Only the backend launch in
`/usrdata/fpv/fpv_gnd_run.sh` was replaced, preserving radio bootstrap, other
services, arguments, library path, directory, umask and priority. The helper
checks artifact hashes, parses configuration as data rather than sourcing shell
code, starts the proxy and waits up to five seconds for readiness. Disabled,
invalid, missing or corrupt configuration/artifacts, an occupied port or readiness
failure select the stock path **before backend launch**. A lock rejects duplicate
backend launches. There is no automatic runtime fallback.

| Installed setting | Recorded value |
| --- | --- |
| Enabled | 1 |
| Host RTP destination | `192.168.3.222:50002` |
| RTP payload bytes | 1200 |
| Listener / upstream port | 50001 / 50000 |
| Tap queue / client limit | 128 / 16 |

Per-boot state under `/tmp/ascent-groundstation/` records mode/reason, process
IDs, integrity/readiness logs and SDP. Statistics are written on orderly exit.
Boot verification confirmed the persistent shim mapping, environment, launcher
hash and three artifact hashes. The VTX was off: missing SDP was expected, and
an initial verifier failure caused solely by requiring that file was corrected.
The next day's result below supplies the later video evidence.

Disabling export or restoring the saved launcher takes effect through a full
receiver reboot. Re-running the startup script beside active services or stopping
the proxy while the backend depends on it is not the validated rollback path.

## Direct viewer

Live playback decoded original HEVC on the host, without transcoding. RTP uses
UDP 50002 and reserves adjacent UDP 50003 for RTCP. The viewer helper checks both
ports, validates destination/payload/codec fields in SDP, and restores USB host
addressing after enumeration. A receiver refresh verifies proxy state and running
destination; cached-only playback assumes those settings remain unchanged.

With the VTX off, existing valid codec parameters can be retained; otherwise a
generic SDP waits for in-band headers. A mid-GOP join can still produce initial
missing-reference warnings. The tested viewer used one decode thread and a
256 KiB receive-buffer request; excessively small buffers produced corruption.
These settings did not establish a measured latency advantage over HDMI.
The [latency comparison](video-latency-comparison.md#the-verified-ascent-exporter-boundary)
identifies a separate source-level delay: normal picture release waits for
classification of the next picture's first-slice NAL, which itself waits for
a following start code. A repeated synthetic probe verifies that dependency,
not its live duration. Stock TCP forwarding does not wait for this assembler.

## Video loss and recovery

On October 7, after approximately 14 hours uptime, the VTX was on but video was
absent. All eight proxy sessions and vendor processes were alive. A bounded
capture found identical control bytes on both sides of the proxy, no reconstructed
TCP gaps or capture drops, and no radio video messages. Status reported
connected=true, MCS=−1 and quality grade 0; radio logs showed retransmission
holds and synchronization unlock/clear events. Backend state retained 1080p60
configuration but showed no encoder-running flag and zero video receive activity.

This located the failure before HEVC/RTP export without identifying the exact
stock radio/control cause. A full receiver reboot preserved pairing, saved
configuration and the persistent installation. Subsequent evidence showed:

- Proxy mode and installed integrity checks passed; codec SDP was generated.
- The VTX encoder-running flag became true.
- A raw receive counter advanced by 7,718,250 over approximately three seconds.
  This is traffic activity, not decoded frames.
- The decoded-picture counter advanced from 16,061 to 16,242: 181 pictures,
  with a decoder estimate of 60.0024 fps.
- The direct host viewer displayed changing pixels. An initial missing-reference
  warning did not recur in the checked log.

This supersedes the earlier VTX-off verification's unresolved video check.
Receiver decode progress and host playback do not independently prove the
physical HDMI display. The incident also motivated rejecting RF-dependent
telemetry when MCS is negative even if the connected flag remains true.

## Artifact identity and limits

| Artifact | SHA256 |
| --- | --- |
| Passive reconnect pcap | `9d648893f9376f4435c461cc5a72221f97a51d339aae9db65c907b13131a695c` |
| Extracted HEVC | `e47efc12b17a9af3b76fae8720e87dbe3da555707d0ef948f0f3b0d97a2e5efa` |
| Proxy executable | `08094ec1300d39d4870f7836cb883ed224ab474f4dca627574dc84fa72151c1c` |
| Redirect shim | `60a99aa6b1f2368f9fdee0aa11b5b24030e1e86fd1aa31ddd7e28066a0ea5985` |
| Startup helper | `94d442e419590b720ae47f52c1d4db06db3b2aec5a5191e4d4ff89e97167ef88` |
| Original launcher | `84775bf56f2a4430e6bc813e41c48709418cf4530c7813e2163a1da772fedfe6` |
| Patched launcher | `861170e3470e204f02003c326e25f3345394d91301c3077ed5158bf39aa5a36c` |

Recorded validation included protocol/RTP checks, ten socket integration cases,
seven BusyBox-ash boot-integration cases and sanitizer runs. These are historical
implementation results, not tests rerun during documentation consolidation.

The deployment exports one selected HEVC stream to one fixed IPv4 destination.
RTSP/RTCP negotiation, dynamic subscriptions, VTX selection, full control API,
runtime failover, HUD/menu export and untested multi-layer/tiled modes remain
outside this result. Long-run recovery reliability and quantitative end-to-end
latency remain open.

The separate [browser-interface study](ascent-browser-interfaces.md) documents
why the tested custom-WebUSB/HEVC viewer remained unvalidated despite working
native host playback. Web Serial file uploads use a different existing interface.

## Unimplemented selection and control design

The original configuration contained two remembered peer entries and a
`save_candidate_position` of 2. This did not establish two reachable VTXes or
an active-peer selector. A daemon work ID identifies a radio device, not a
remote video feed. Control needs separate remembered, detected/last-seen,
requested and actually streaming states.

The proposed switch sequence was:

```text
STREAMING(A) → SWITCHING → ASSOCIATING(B) → WAIT_CODEC_CONFIG/IDR → STREAMING(B)
```

Old buffered data must not masquerade as the new peer: the design calls for a
new stream epoch/SSRC, refreshed codec configuration and host decoder reset.
Association completion is distinct from command acceptance and picture readiness.
This state machine is a design requirement, not an implemented or measured
seamless-switch feature.

A proposed request/reply agent would own selection and export controls and
coordinate with the stock GUI, which can independently reapply settings. A
serial frontend would use that same owner. The existing `/dev/ttyGS0` updater,
`ttyS0` console and backend-owned `ttyS4` cannot simply be assumed spare command
channels. A second ACM function requires its own endpoint/rebind validation.
The deployed proxy has no such network/serial command service.
