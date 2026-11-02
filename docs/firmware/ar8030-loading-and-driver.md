# AR8030 loading, transports and driver lifecycle

[Firmware](README.md) · [Host-link protocol](../protocols/ar8030-host-link.md)

This account covers the examined arlink implementation and matching stock
USB/SDIO observations. Load addresses and section sizes below identify one
captured image; other firmware profiles must be parsed and checked separately.

## Firmware container

The examined radio image begins with a 64-byte header:

| Offset | Encoding | Meaning |
| --- | --- | --- |
| `0x00` | LE32 `0x41528030` | Magic, bytes `30 80 52 41` |
| `0x04` | LE16 | Observed version 1 |
| `0x06` | LE16 | Header size 64 |
| `0x18` | Two LE32 values | Runtime load address and length |
| `0x20` | Two LE32 values | SPL load address and length |
| `0x28` | Two LE32 values | Embedded configuration load address and length |

SPL, configuration and runtime follow in that order, each on a 512-byte boundary.
For the studied `bb_demo_cx485_2PA.img`, section offsets were `0x200`, `0x600`
and `0x4600`. The separate JSON configuration overwrites the embedded
configuration during loading; it is an explicit input to the running firmware.

## USB boot ROM

The boot ROM enumerates as `4152:8030`, with a HID-class interface and no
non-control endpoints. Upload uses vendor OUT control requests:

```text
bmRequestType = 0x40
bRequest     = 0x0c
wValue       = 0
wIndex       = 0
```

Each request carries:

| Bytes | Field |
| --- | --- |
| 0–1 | ASCII `US` |
| 2–3 | LE16 data length, at most 4084 |
| 4–7 | LE32 destination address |
| 8–11 | Zero |
| 12 onward | Data |

The recorded sequence, approximately 122 requests in 100 ms, was:

1. Send the 64-byte image header to `0x002f0040`.
2. Send the 944-byte SPL to `0x0a098a00`.
3. Send the embedded 16 KiB configuration to `0x00200000`.
4. Send the runtime to `0x00204000`.
5. Send the separate JSON to the configuration address, unpadded.
6. Send a zero-length block to address zero to run.

There were no reads/status exchanges between those blocks. The final request
can fail as the device disconnects; successful re-enumeration, not that return
code alone, establishes the transition to running firmware.

The running USB identity is `1d6b:8030`, with bulk OUT `0x01`, bulk IN `0x81`
and 512-byte endpoint packets. Logical framing remains the
[mixed-endian host protocol](../protocols/ar8030-host-link.md).

## SDIO boot and runtime

SDIO uses function 1, IDs `4152:8030` for ROM and `4152:8031` after loading.
Upload follows the same logical section sequence but uses an `SD` prefix and
function-1 FIFO at address zero. Transfer sizing follows 512-byte block rules;
runtime frames longer than one block are padded to whole blocks.

| Function-1 register | Meaning |
| --- | --- |
| `0x13` | Pending state; bit 4 indicates events |
| `0x14` | Interrupt enable; probe writes 1 |
| `0x54` | Mailbox 0 |
| `0x58` | Mailbox 1 |
| `0x5c` | Available receive blocks |
| `0x60` | Available transmit blocks |
| `0x68` | Events: bits 0/1 mailboxes, bit 2 receive data, bit 3 transmit room |

Probe writes `0xf0` to the event register. Receive frames begin at block
boundaries; bytes beyond the logical frame in the final block can be stale.
Transmit waits for enough room for the whole frame. Splitting a frame across
transfers caused drops in the studied transport.

After upload, host polling detects removal and the new running ID. A generic
reset-on-re-enumeration path can erase newly loaded RAM firmware; board-specific
power sequencing must preserve the intended transition.

The stock P1 SDIO block-tail envelope is an additional compatibility mode, not
native arlink framing. The P1 source stack uses native framing and a matching
radio image. Replacing a stock driver on an already running chip can leave it
silent after function disable/re-enable; cold-ROM initialization provides the
tested starting point.

## Device-file ownership

The examined driver exposes one exclusive `/dev/arlinkN` instance per chip:

- A read returns one complete logical frame. A small buffer returns `EMSGSIZE`
  and leaves that frame queued.
- A write supplies one frame and waits for transport acceptance, with a one-second
  timeout. Socket-level completion is a separate protocol exchange.
- A second open returns `EBUSY`. Frames arriving while closed are discarded.
- On removal, queued frames can drain before `ENODEV`; writes/new opens fail.
  The recreated device may reuse the same pathname while an old descriptor is
  still permanently disconnected.

The driver tracks successful socket open, init and start responses. On close it
closes owned sockets, sends STOP (`0x0b000001`) and DEINIT (`0x0b000003`) when
needed, and bounds each cleanup exchange. Without the final lifecycle steps,
repeated forced application exits could wedge the USB OUT endpoint.

Cleanup uses message ID zero and suppresses late timed-out cleanup replies for
ten seconds, preventing them from reaching the next owner. It stops trying after
the first failed cleanup operation rather than indefinitely retaining the device.

## Receive queues and attached sockets

The USB implementation keeps eight 8 KiB receive transfers in flight. A framer
checks markers, length and header XOR before queueing complete frames. Defaults
are a 1 MiB main queue and 512 KiB per attached socket, using virtually contiguous
allocations rather than requiring a large physically contiguous block.

An attached socket routes its data directly to a separate descriptor, bypassing
the library reader-thread ring. Each record includes a monotonic timestamp from
the receive completion and the payload. This timestamp identifies host reception,
not the sender's camera exposure or on-air departure.

Applications attach before opening a radio socket to avoid initial data being
routed to the wrong queue. They can poll multiple attached sockets together;
serial timeout waits on empty sockets otherwise add avoidable video delay.

After 50 consecutive receive errors or a stalled IN endpoint, the inspected USB
driver marks the device gone and schedules reset/re-enumeration. Applications
must reopen; retrying a dead descriptor cannot attach it to the new device.

ROM upload retries are also bounded: four uploads without an intervening
30-second running period stop further attempts until driver reload. Missing
files or rejected preflight bounds do not count as completed uploads. These
thresholds describe this driver implementation, not requirements imposed by
the radio hardware.
