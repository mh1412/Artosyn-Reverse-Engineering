# VR04 reference AR8030 SDIO and IP transport

[Protocols](README.md) · [Reference board](../devices/betafpv-vr04-reference.md)

## Scope

This is the MissingLynk reference `artosyn_sdio` transport described in kernel
snapshot `e3885c6394a7025eacd3a23f473276d9823beb25`. It combines an SDIO firmware
loader, character-device control and an IP netdev. It is distinct from the
[Ascent socket-oriented host protocol](ar8030-host-link.md) and from RF framing.
The original research reports live reference-board video; this consolidation
does not rerun it or claim another board's compatibility.

## Host, reset and boot identities

The reference host is DesignWare MMC at `0x01b00000`, GIC SPI 48, with four-bit
SDIO and GPIO23 active-low radio reset. GPIO and clock glue must precede the
radio driver. The removable SD host is a separate instance at `0x01c00000`;
runtime `mmcN` numbering can differ from DT instance names.

| SDIO identity | Interpretation |
| --- | --- |
| `4152:8030` | ROM loader; firmware upload needed |
| `4152:8031` | Firmware-running function; construct control/netdev interfaces |
| Observed `4152:2a22` | Unmatched corrupt identity; no driver probe or upload |

The corrupt-ID case survived GPIO reset; a cold power cycle recovered it. The
reference DT exposed reset but no independently controlled radio supply, so no
software power-cycle mechanism was established. Repeated/longer reset behavior
remained untested.

A separate failure was a pre-load wait that accepted only `8030` even when
firmware survived a host reboot and the chip enumerated as `8031`. The later
documented correction accepts **both IDs**, matching the driver. This supersedes
the same report's earlier statement that every bring-up necessarily reloads the
firmware. A retained image and a corrupt identity are different states.

Old claims that GPIO directions were universally unreadable also do not survive
the later [Gnd485 readback investigation](../soc/ar9311-platform.md#direction-readback-correction).
The reference's GPIO23 assignment remains board-specific.

## Firmware upload and host mailbox

The radio inputs are `bb_demo_gnd_d.img` and `bb_config_gnd.json`; `chagall.bin`
is the separate video-codec firmware. The reported ROM transfer uses `SD` magic
`0x4453`, a 64-byte SPL header at `0x2f0040`, up to 4084 payload bytes plus a
12-byte header per packet, and an address-zero/length-zero finalize record.
Configuration JSON is streamed verbatim. Re-enumeration as `8031`, netdev creation
and roughly 130 mA additional current were recorded bring-up signals; current
draw alone does not prove useful radio data.

| CMD52 register | Meaning in the examined driver |
| --- | --- |
| `0x13` | Status; busy bit 4 |
| `0x14` | Control |
| `0x44`, `0x48` | TX mailbox |
| `0x54`, `0x58` | RX mailbox |
| `0x5c` | RX count, in 512-byte blocks |
| `0x60` | TX credit, in 512-byte blocks |
| `0x68` | Event status: mailbox0/mailbox1/RX/TX in bits 0–3 |

The report records initialization writes `0x14←01`, `0x68←f0`. Control is exposed
through `/dev/artosyn_sdio` with seven `'v'` ioctl operations: command, register
read/write, message get/post and CCCR read/write. The command encoding is
`0xc0207600`. This is a host control ABI, not an RF command-number table.

## IP data and receive resynchronization

The netdev has MTU 4096 and `IFF_NOARP | IFF_BROADCAST`. Each run's final
512-byte block ends with a four-byte trailer:

| Byte | Meaning |
| --- | --- |
| 0 | `00` |
| 1 | Type: `cc` IP tunnel, `dd` command/response, `ee` flow-control ACK |
| 2–3 | Little-endian length |

The IP tunnel carries a 12-byte compressed IPv4 header. The driver reconstructs
the ordinary header using the interface's address and the transported final
address octets for the point-to-point `10.0.0.x` link. Interface-address changes
are observed through an inetaddr notifier. This summary does not assign an
unreviewed byte-by-byte schema to all 12 bytes.

A naive search for `45` inside HEVC data mistook payload for IPv4 and fabricated
ghost destinations. The corrected receive path carries split headers between
reads, checks UDP/TCP/ICMP/IGMP protocol plausibility and total length ≤4352,
and back-parses multi-run transfers using their trailers. Junk skips increment
`rx_frame_errors`. CMD53 block size must be forced to **512 for both identities**:
the re-enumerated `8031` is a fresh function whose default otherwise desynchronized
the FIFO and left receive counters at zero.

The reference report records approximately 1.6 MB/s through the IP/UDP stack
with zero `InAddrErrors`/`ReasmFails`, and a separate roughly 1.5 MB/s sustained
video observation. Polling, radio TX-power state and the UDP port-10000 media
handshake gate the port-10001 video stream. Association alone is insufficient.

Netdev byte counters establish traffic, not correct pictures or optical latency.
GET-side telemetry could remain unchanged on a dead link, and a separate apparent
SDIO fault was traced to air-unit battery brownout. Warm module unload/reload
hung the reference device, so its validation used fresh RAM boots. These are
recorded failure boundaries, not generalized recovery guarantees.

The [system video path](../systems/video-paths.md#betafpv-related-airground-pipeline)
describes the tiled HEVC application layer above this transport.
