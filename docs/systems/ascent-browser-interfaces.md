# Ascent stock Web Serial management

[Systems](README.md) · [USB management](../protocols/ascent-usb-management.md) ·
[Native ground station](ascent-groundstation.md)

Several browser investigations used the standard Gnd485 receiver.
Web Serial reused its existing CDC ACM management interface for file uploads.
## Early Web Serial file uploader

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
