# Ascent browser interfaces: serial uploads and proposed USB video

[Systems](README.md) · [USB management](../protocols/ascent-usb-management.md) ·
[Native ground station](ascent-groundstation.md)

Two different browser investigations used the standard Gnd485 receiver.
Web Serial reused its existing CDC ACM management interface for file uploads.
The proposed WebUSB video path required a new vendor bulk function and a usable
HEVC decoder. Success at the serial interface did not resolve either video
requirement.

## Web Serial implementation and validation

The browser uploader constructs the same OTRA packets as the Python client.
Its implemented upload command set is information `0x3c`, optional preparation
`0x72`, metadata `0x73` and data `0x74`. It exposes no FileEnd, update-progress or
clean-mode operation. The service is accessed through the existing USB serial
port, without SSH, receiver network setup or a new kernel USB function.

The application filters the browser port picker to `1d75:0101` and requires
the reported model `Ascent_VRX` before upload. Permission applies to a device
attached to the browser's computer. Its local HTTP server only serves a fixed
asset whitelist; it does not open serial ports, receive uploaded file contents
or become a remote serial bridge. The tested browser environment used secure
localhost; the application also supports HTTPS serving. A plain LAN HTTP URL
does not become secure merely because the server listens on all interfaces.

The implementation streams the selected file twice: first for metadata MD5,
then for acknowledged chunks, recalculating MD5 during transmission. Binary and
empty files are supported. Default chunks are 64 KiB, bounded by the advertised
device limit and a 1 MiB maximum. File size must fit the signed 32-bit field.
Destinations are absolute printable ASCII filenames of at most 127 bytes, with
no trailing slash or final `.`/`..` component.

Preparation is **enabled by default in the browser UI**, unlike the Python
copy client's default. It resets retained receiving counters and may affect normal
receiver operation. The background read pump bounds buffers, allows one request
at a time, handles fragmented reads and matches sequence/command/CRC. DTR and
RTS are explicitly deasserted. Disconnect cancels reads and releases stream locks.

Success means every byte was acknowledged with status OK and exact counters,
followed by unchanged identity/firmware. It is not independent filesystem
readback, remote whole-file MD5 acceptance or guaranteed durability. Timeout,
cancellation, corrupt/rejected responses and read failures stop without uncertain
write retries. Once metadata has been sent, the destination may be truncated or
partial. The downloaded audit contains names, checksums and command summaries,
not the file contents.

### Separate normal reboot

The reboot action checks identity and sends command `0x03` with a fixed 32-byte
payload: `normal` plus 26 zero bytes. Other payloads, including `clean`, are
rejected by its packet constructor. It is disabled during upload and does not
run automatically after copying.

The inspected 17.5.3 handler sends a 68-byte status ACK before its normal branch
invokes `/etc/sys_reboot.sh`; the reference helper syncs before watchdog reset.
The browser releases the port afterward and requires reconnect. A disconnect
after a complete request write but before ACK is recorded as sent/ACK-unverified,
not as verified boot completion. Failed writes, timeout, bad ACK and rejection
remain distinct failures, without automatic reboot retries.

The recorded validation included 14 JavaScript tests, 123 Python tests and an
isolated real Chromium renderer connected to a **fake serial receiver**. Packet
bytes were compared with Python; tests covered fragmented transfers, limits,
stale/corrupt ACKs, cancellation, lock release, audit content and reboot behavior.
No physical browser-to-VRX upload or reboot was performed during that work.
The [Python hardware experiments](../experiments/ascent-file-upload.md) establish
the underlying device path, not end-to-end browser qualification.

## Custom WebUSB video: receiver-side findings

The October 6 study inspected Linux 4.9.38 and DWC2 controller `8000000.usb`,
enumerated at USB 2.0 high speed. The existing composite gadget was:

| Function | Interfaces | Allocated endpoints |
| --- | --- | --- |
| RNDIS | 0–1 | IN 1 bulk, IN 2 interrupt, OUT 1 bulk |
| CDC ACM | 2–3 | IN 3 bulk, IN 4 interrupt, OUT 2 bulk |

The controller reported ten device endpoints, dynamic FIFOs and multiple TX
FIFOs. IN 5 and OUT 3 had no queued requests or transferred data. Spare endpoint
numbers were only a feasibility clue: unused TX FIFOs 5–10 were **16 words**
each, or 64 bytes, less than a 512-byte high-speed bulk packet.

The reported FIFO budget was 2,984 words, with 2,720 allocated. Enlarging one
16-word FIFO to 128 words would consume another 112 of the remaining 264 words.
This arithmetic suggested room; no new allocation, composite binding or
throughput was demonstrated.

Temporary **unbound** configfs gadgets produced:

| Allocation probe | Result |
| --- | --- |
| `ffs.ascent_probe` | ENOENT |
| `gser.ascent_probe` | ENOENT |
| `SourceSink.ascent_probe` | ENOENT |
| `acm.control` | Successful positive control |

FunctionFS was absent from the observed filesystem inventory; matching symbols
and modules were not found in the inspected locations. Generic vendor scripts
contained FunctionFS/ADB and SourceSink branches, but those branches did not
establish support in this kernel. `/proc/config.gz` was unavailable and a matching
loadable-module build environment was not established. Module loading being
enabled did not prove ABI compatibility for an added function.

All temporary gadgets were removed without assigning a UDC. The original `g1`
binding, proxy mode and backend/daemon/UI process IDs remained unchanged. No
existing RNDIS or ACM interface was detached or claimed by the browser.

## Browser HEVC results

The captured stream's SPS identifies `hvc1.1.6.L123.A0`, Main profile, level 4.1,
1920×1080. Independent FFmpeg inspection agreed on those parameters. Archived
capability tests used separate profiles on the Linux host:

| Tested build | Modes | WebUSB/WebCodecs APIs | Actual HEVC configuration | H.264/VP8 controls |
| --- | --- | --- | --- | --- |
| Chromium 153.0.8010.36, direct executable | Headless and desktop | Present | Unsupported | Supported |
| Chromium 154.0.8037.57, direct executable | Headless and desktop | Present | Unsupported | Supported |
| Brave, reported version 154.0.8037.93 | Headless and desktop | Present | Unsupported | Supported |

Both `hvc1` and `hev1` forms failed with no-preference, hardware-preferred and
software-preferred modes. Additional Main/Main10 configurations also failed.
Actual browser HEVC decoding was skipped, so this is not a decode-rate result.
Initial Chromium Snap-wrapper launches failed to navigate to localhost with
`ERR_ACCESS_DENIED`; later direct-executable tests reached the probe and supplied
the separate HEVC result.

These are historical build/host observations, not a claim about all current
browsers or operating systems. API presence, device permission, interface
claimability and actual codec support are separate requirements. A software/WASM
decoder would need its own rate and latency evidence.

## Proposed ARV1 record

The probe implemented a test framing format, **not a deployed VRX protocol**:

| Offset | Bytes | Field |
| --- | ---: | --- |
| 0 | 4 | ASCII `ARV1` |
| 4 | 4 | LE32 payload length, 1 through 4 MiB |
| 8 | 4 | LE32 flags: 0 or 1, bit 0 marks key picture |
| 12 | 8 | LE64 timestamp, microseconds |
| 20 | Variable | One Annex-B HEVC access unit |

The streaming reader accepts arbitrary read splits and rejects invalid magic,
length, flags or timestamps above JavaScript's maximum safe integer. The record
does not contain a checksum, stream-epoch field or a hardware capture-time clock
mapping. Those must not be inferred from the timestamp label.

A JavaScript roundtrip preserved 1,481,737 payload bytes across 315 complete
pictures using 511-byte input chunks. The final unfinished picture was excluded.
This was an in-memory/HTTP-fixture test, not USB throughput or successful browser
video decoding.

## Remaining end-to-end evidence

A completed custom-bulk viewer still needs an actual supported HEVC decode,
a usable receiver bulk function with appropriate FIFO allocation, successful
composite enumeration alongside management functions, deterministic byte-transfer
tests and live video validation. Short packets, zero-length completion, backpressure,
paused readers, disconnect/reconnect and loss recovery need measured behavior.
The proposed video copy queue must remain bounded and independent of stock
forwarding, as in the native ground station. No end-to-end browser video result
was established by these probes.
