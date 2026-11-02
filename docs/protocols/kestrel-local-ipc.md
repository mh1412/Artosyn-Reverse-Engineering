# Kestrel backend and passive-HUD IPC

[Protocols](README.md) · [Software UI architecture](../systems/kestrel-software-ui.md)

Kestrel's development architecture separates backend state/operations from UI
ownership and passive HUD rendering. Its two local protocols have different
headers and ownership rules. Neither is the stock radio transport or an
over-the-air protocol.

## Backend state and commands: KGND v1

The backend uses local `AF_UNIX/SOCK_SEQPACKET`, mode 0600 and matching-UID peer
credentials. A separate lock is acquired before opening the platform adapter;
a second instance cannot unlink the live socket. Shutdown removes only the
instance's own socket inode.

All multibyte fields are little-endian:

| Offset | Width | Field |
| --- | ---: | --- |
| 0 | 4 | ASCII `KGND` |
| 4 | 2 | Version 1 |
| 6 | 2 | Type: 1 Hello, 2 State, 3 Command, 4 Reply |
| 8 | 8 | Sequence |
| 16 | 4 | Payload length |
| 20 | Variable | Explicitly serialized payload |

Maximum complete packet size is 16,384 bytes. No native structures, pointers or
file descriptors cross this interface. Floating-point values use explicit IEEE
bit serialization with finite-value checks; enum, Boolean, string and length
constraints are checked independently.

| Message | Contract |
| --- | --- |
| Hello | Complete initial state and service session ID; header sequence zero |
| State | Complete snapshot; payload/header publication sequences agree |
| Command | Client-local increasing request ID and typed operation |
| Reply | Matching request ID, outcome/detail and optional bounded catalog page |

Snapshots publish at 10 Hz independently of drawing. Each peer has one coalesced
pending snapshot and a bounded reply queue; peer count and work are bounded.
A subscriber that makes no progress expires after five seconds. The client
clears unavailable backend/flight state after two seconds without service
progress and resynchronizes from a new session after reconnect.

Original monotonic telemetry timestamps survive serialization. An IPC heartbeat
does not refresh old voltage or arm state. Pending commands fail explicitly on
an uncertain disconnect and are never automatically replayed. Menu state and
local preferences remain owned by the UI.

Commands cover refresh, recording catalog, recording/playback, pairing and typed
radio/camera settings. Capability gates distinguish an implemented command type
from an available operation. The `ascent-stock` adapter only provides telemetry
and an optional catalog; hardware controls return `unsupported`.

The read-only catalog is rooted at a service-owned directory descriptor. It
accepts basename IDs, excludes hidden/non-video files, directories and symlinks,
and bounds scan size and pagination. Overflow is reported rather than silently
presenting a partial catalog. Client-supplied filesystem paths are not followed.

## Passive renderer handoff: KHUD v3

The UI and child HUD use a private `SOCK_SEQPACKET` socket pair. The source
encoder confirms a **16-byte header with no payload-length field**; packet
boundaries provide the length. This differs from KGND's 20-byte header.

| Offset | Width | Field |
| --- | ---: | --- |
| 0 | 4 | ASCII `KHUD` |
| 4 | 2 | Version 3 |
| 6 | 2 | Type: 1 Hello, 2 Setup, 3 Data, 4 Frame, 5 Release |
| 8 | 8 | Sequence |
| 16 | Variable | Message-specific payload |

Maximum complete packet size is 8,192 bytes. Truncated packets/ancillary data
and more than one received descriptor are rejected; received descriptors are
close-on-exec. Setup transfers one shared-memory descriptor using `SCM_RIGHTS`.
Pixels do not travel through the socket.

| Message | Direction and meaning |
| --- | --- |
| Hello | Worker proposes cropped surface layout |
| Setup | UI accepts validated geometry and supplies two shared slots |
| Data | UI supplies a complete display-ready snapshot; worker coalesces obsolete poses |
| Frame | Worker supplies slot, data sequence, changed-region mask and render/cache counters |
| Release | UI acknowledges the exact slot/sequence lease after synchronous drawing |

The layout payload starts with five u32 fields: viewport width, height, format
code 1, slot count 2 and region count. Each region is five u32 fields: x, y,
width, height and role. Format 1 means tightly packed ARGB8888, straight alpha,
stride `region_width × 4`. Valid viewports are 640–4096 by 360–2160; one to eight
nonempty in-bounds regions must fit an aggregate one-full-screen pixel budget
per slot. Roles are generic HUD=0, left wing=1, right wing=2 and status=3.

Frame payloads contain u32 slot, u64 render microseconds, u64 cache bytes,
u64 raster count and u32 changed-region mask. Slot must be 0 or 1. The Data
serializer bounds text to 128 bytes without embedded NUL, normalized finite
fractions to 0–1, metrics to 24 unique valid IDs, and scale labels to six per
metric. Version 3 adds renderer-neutral metrics and a simulation flag; the
earlier version 2 checkpoint introduced region damage masks.

## Buffer lifetime and recovery

The producer never overwrites a leased slot. The UI keeps its front slot until
synchronous composition finishes and drops decoder/image-cache references before
release. A sealed memfd bounds capacity where supported; an unlinked owner-only
file is the compatibility fallback. These are CPU-sharing contracts, not proof
that the same memory is importable by GE2D.

HUD failure clears its layer and starts bounded restart/resynchronization while
the UI continues input and preference editing. Parent death and normal shutdown
stop/reap the child. Gallery scenes suppress the layer immediately and pause
drawing with a disabled snapshot, independently of worker latency.

Host tests covered schema truncation, malformed fields, descriptor ownership,
pixel equivalence, slow/failed peers, command uncertainty and restart behavior.
The physical [UI/HUD experiment](../experiments/kestrel-ascent-ui.md) also killed
the worker during a theme preview without stopping the UI or stock decoder.
Native radio/media adapters and accelerator completion semantics remain separate
validation tasks.
