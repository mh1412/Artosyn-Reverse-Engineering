# Open8030 runtime and diagnostics

[Firmware](README.md) · [AR8030](../soc/ar8030.md) · [Validation](../experiments/open8030-validation.md)

Open8030 executes on the AR8030 radio processor. In the matched Ascent system,
the application host still runs the Linux SDIO driver, radio daemon, video
backend and UI. The runtime described here developed through the receiver
reconstruction; later AP support has its own board-specific loading requirements.

## Boot and memory

The generated container combines a source-built bootstrap, a configuration
reservation and the radio runtime. It does not require a stock container as a
packaging template. Hardware builds still require the matched board profile and
external loader configuration.

| Component | Container offset | Destination | Recorded release size |
| --- | --- | --- | --- |
| Header | `0x0000` | Loader metadata | 64 bytes, padded to 512 |
| Bootstrap | `0x0200` | `0x0a098a00` | 792 bytes, padded to 1,024 |
| Configuration | `0x0600` | `0x00200000` | 16 KiB zero-filled reservation |
| Runtime | `0x4600` | `0x00204000` | 115,332 bytes, padded to 115,712 |

The 2026-10-22 validated container is 133,632 bytes. The loader replaces the
configuration reservation with external JSON. The bootstrap also uses temporary
RAM in this region: load addresses do not imply independently resident objects
or determine execution order.

The recorded runtime uses 688,272 of 704,512 reserved RAM bytes, leaving 16,240
bytes in the build's accounting. This is not a worst-case stack or DMA margin.
Earlier builds recorded a 66,792-byte runtime and 670,816-byte RAM use; those
figures do not describe the release above. Exact artifact identities are in the
[validation record](../experiments/open8030-validation.md#artifact-identities).

The bootstrap performs low-level setup and the loader reset handshake. The
runtime initializes transport, sockets and events; validates configuration;
probes hardware and identity; reads factory data; applies pins and defaults;
calibrates; sets up PSRAM; and prepares the radio. On the matched DEV path,
preparation holds RF until the Linux backend supplies its candidate list.

The reset hook initially disables caches and interrupt delivery. The later
hardware runtime enables instruction/data caches, while CPU interrupts remain
masked. A single cooperative loop polls RF and SDIO; Zephyr multithreading and
the system clock are disabled. The UART-based QEMU build exercises framing and
dispatch without emulating radio hardware.

## Queues, scheduling and ownership

Outbound events take priority, followed by ready close replies. Socket read data
and write completions then alternate. RF polls surround copying and framing,
and one SDIO transmit entry is reserved for host replies.

| Resource | Compiled bound in the examined receiver runtime |
| --- | --- |
| Host-frame payload | 4,096 bytes |
| SDIO transmit queue | Four entries |
| Host socket ports | Four |
| Socket receive storage | Up to 56,320 bytes per port; active capacities can be smaller |
| Pending host writes | Two per port |
| Host event queue | 16 descriptors sharing 4,096 bytes |
| Individual event body | Up to 1,024 bytes |

RF receive bytes leave a socket queue only after SDIO accepts a framed copy.
Host write bytes commit when RF transport owns its copy. A successful write
completion therefore does not acknowledge reception by the peer.

Closing a socket stops new endpoint transfers. A partially copied active write
completes with `-10`; queued writes that have not started are discarded. Cleanup
precedes the close reply, which is retried until accepted by SDIO. The port cannot
reopen before that acceptance. Already submitted RF/SDIO copies retain their own
storage. These copy-mode rules do not establish equivalence to the stock RTOS
workers and retained-descriptor ownership.

Stock host ports 0 and 1 already have internal stream owners in the matched
configuration. Native admission of those ports reflects missing owners, rather
than extra compatible functionality. In mode 3, internal port 0 carries
UART1/SBUS; port 1 connects to a DEV event bridge or AP UART2. Implementing a
reserved-port rejection alone would not provide these services.

All twelve recovered event subscriptions and their declared sizes are supported
by the copied event transport. The examined runtime integrates producers for
link/MCS events 0/1 and the selected C8 receive bridge, event 8. Other accepted
subscriptions need their original producers and routes. Unsubscribing does not
revoke an already queued event copy.

## Diagnostic transport

The Linux daemon translates some requests and message IDs; its TCP endpoint is
not a transparent chip connection. Open8030 diagnostics use direct chip IDs
`0x0c5a....` and selected daemon aliases `0x015a....`. These are development
interfaces, separate from the [vendor host protocol](../protocols/ar8030-host-link.md).

The matched daemon provides its encoder a 4,096-byte buffer including the
19-byte frame overhead, leaving at most 4,077 payload bytes. The complete
4,084-byte RX diagnostic exceeds that independent limit. Increasing the radio's
SDIO buffer cannot fix this daemon constraint.

### Bounded RX record

Alias `0x015a0017` requires four request bytes: LE16 offset followed by LE16
count, with count at most 1,024. Read the 4,084-byte record in ranges. Empty or
malformed requests and out-of-range windows return `-22` without a payload;
zero-count reads through the end offset are valid.

Direct ID `0x0c5a0017` also accepts ranges and retains the empty-request form
for the complete record. Successive ranges are live reads, not an atomic snapshot.

### SDIO and socket flow record

Alias `0x015a002a`, direct ID `0x0c5a002a`, takes an empty request and returns
220 bytes: 55 little-endian 32-bit words.

| Word indices | Contents, in order |
| --- | --- |
| 0–2 | Version 1, free SDIO TX entries, active descriptor age in ticks |
| 3–14 | SDIO polls, host enables, RX packets, TX descriptors started, errors, invalid RX, TX backpressure, TX queue high-water mark, descriptors finished, last finished duration, maximum finished duration, padded wire bytes started |
| 15–24 | Port 0 record |
| 25–34 | Port 1 record |
| 35–44 | Port 2 record |
| 45–54 | Port 3 record |

Each ten-word port record contains queue high-water bytes, largest fragment,
drop-call count and last dropped byte count, followed by the SDIO state at that
drop: free entries, active age, last duration, maximum duration, descriptors
started and RX packets. Unused drop snapshots are zero. Records survive socket
close/reopen until firmware restart and reads do not clear them. Combine them
with current capacities and counters from diagnostic `0x015a001a`.

The matched machine timer runs at 40 MHz: 40 ticks per microsecond. Durations
include host waiting and polling delay; they are not isolated DMA transfer times.
Aborted descriptors also count as finished, and wire bytes count starts rather
than delivery. Unsigned 32-bit counters wrap. A descriptor stall lasting a full
timer period, about 107 seconds, cannot be measured unambiguously this way.

### Transport telemetry record

Alias `0x015a002b`, direct ID `0x0c5a002b`, takes an empty request and returns
596 bytes: twenty LE32 words, a 260-byte last-C8-frame area and a 256-byte
most recent host-data fragment. Lengths delimit valid bytes; unused storage is
zero. Counters separate input, valid frames, publication, no subscription,
event drops and malformed data.

Reads neither consume stream data nor create a subscription. Samples can be
truncated or missed between reads, so this is not a lossless capture. Link
changes clear partial framing and sampled data while preserving counters.

The C8-to-event-8 bridge transports complete envelopes. MAC port-3 application
data continues through host socket 2, where Linux decodes transmitter status
and MSP. A temporary radio-side application decoder and diagnostic `0x...002c`
were removed at stage 232. See [application stream formats](../protocols/ascent-application-streams.md).

## Register-read semantics

Vendor getter `0x01000064` accepts LE16 byte offset and LE16 count. The recovered
contract allows unaligned windows, counts 0–256 and end offsets no greater than
4,096. Each output byte performs a separate aligned 32-bit register read, so
changing registers can produce a non-atomic byte sequence.

A well-formed four-byte request returns a 256-byte payload even for range error
`-6`. Open8030 zeroes unused and error bytes. Its additional validation returns
`-22` without a payload for malformed lengths and `-11` before probing; these
guards are not claimed as exact stock behavior. The comparison covered 360
original-instruction cases and selected stock/native hardware requests, not
every register's read side effects.

## Session and status interpretation

The matched VRX loader checks image, software, configuration and SDIO identity
before replacing the running stack in RAM. Its backend hardware IDs are board
485, board version 16 and RF board 16. They are not user-selected product names.

Startup allows 45 seconds for services and then 30 seconds for native identity
and RF link. Successful retention disarms rollback and has no five-minute
expiry. Before retention, a 300-second observation window and independently
scheduled 355-second fallback attempt restoration. Neither guarantees recovery
from a hung kernel or failed driver reload. Preflight can fail before either is
armed. Reboot restores the stock radio on this RAM-loading path.

Session controls are keyed by the supplied host string; hostname and IP address
can select different records. Receiver temporary logs disappear on reboot.
Status requires native identity and link state 2. The fixed identity text
`rf=unsupported` is not a live link field, and `video_verified: false` means the
command did not inspect video. Restoration checks for hashes, enumeration and
services likewise do not establish a visible camera scene.

Black camera areas with live overlays occurred under both stock and replacement
firmware during warm reloads. A full receiver reboot restored the reference
scene in the recorded tests; the cause remains unresolved. HDMI capture must
identify the actual adapter and inspect fresh camera content. Adapter cadence,
link state and advancing counters do not measure RF frame rate or end-to-end
latency.
