# DJI receiver metadata, host framing and recovery contracts

[Protocols index](README.md) · [Receiver architecture](../systems/dji-receiver-comparison.md)

## Evidence boundary

These are static findings from Goggles 3 v01.00.1400 and shared libraries in the
October 12, 2026 study. They describe receiver expectations and available code,
not a captured O4 RF format or a traced air-unit implementation. O4 update
payloads [remained encrypted](../firmware/dji-package-boundaries.md).

## Stream metadata and producer feedback

Receiver codec diagnostics name codec type, dimensions, bit depth, SPS validity,
real frame rate, timestamps/timebase, frame/reference indices, CRC/length and
encoder platform/subversion. These show that the upstream stream has a richer
contract than an unlabelled sequence of compressed bytes. They do not prove that
the air application processor originates every field: modem/receiver translation
may contribute metadata.

Shared producer code handles channel bitrate, per-frame target/actual bits,
QP/PSNR, real bandwidth, codec/strategy restart, optional resolution changes,
GDR/I-frame recovery and old-unsent-frame skipping. ROI/motion options also exist.
Some producer functions serve the goggles' own sharing encoder. Their presence
does not establish every feature's activation in an O4 air unit.

## Recovered `6D` host header

The inspected `libwlm_liveview.so` has SHA-256:

```text
770763576ff8880493368465792a583fab44eacd38bb24e7a92c7fa66910f6ba
```

| Function or data | ELF virtual address | Recorded size |
| --- | --- | --- |
| `libliveview_add_header` | `0x4868` | 860 bytes |
| `libliveview_get_header_len` | `0x5068` | 204 bytes |
| `libliveview_crc_cal` | `0x5960` | 164 bytes |
| Header length table | `0x7b30` | Entries 8, 8, 16 |

These are ELF virtual addresses, not runtime-relocated addresses. Most other
string offsets in this study are file offsets and must not be substituted for VAs.
Header enum 2 selects tag `0x6d` and a **16-byte** header; `6D` is not its length.

| Byte offset | Field | Confidence |
| --- | --- | --- |
| 0 | `0x6d` tag | Recovered serializer |
| 1 | Flags | Complete meaning unmapped |
| 2–5 | Little-endian 32-bit frame counter | Recovered serializer |
| 6–9 | Low 20 bits of a little-endian 32-bit value | Length-like function argument; full schema unresolved |
| 10–14 | Additional fields | Unmapped here |
| 15 | CRC of bytes 0–14 | Recovered and emulated helper |

CRC parameters are reflected polynomial `0x8c`, initial value 0 and final XOR 0.
The recorded pure-function emulation produced the following vectors; an
independent reference calculation during wiki consolidation reproduced them.

| Input bytes | CRC |
| --- | --- |
| Empty input | `00` |
| ASCII `123456789` | `a1` |
| Sequential bytes `00` through `0e` | `21` |

```python
def liveview_crc(data):
    crc = 0
    for value in data:
        crc ^= value
        for _ in range(8):
            crc = (crc >> 1) ^ (0x8c if crc & 1 else 0)
    return crc
```

This helper is host-side framing. It does not identify RF encryption, FEC,
over-the-air headers or which O4 code path calls it. The unmapped bytes should
remain unmapped rather than be filled from superficially similar protocols.

## Negotiation, reassembly and recovery

WLM's capability vocabulary includes 4D/5D/6D slice headers, 6D headers with SSFN,
whole-frame delivery and I-frame-v3 support. Local/peer logical IDs are `0x0e07`
and `0x0907`; the generic `ssfn_6d` default is 0. A capability declaration does
not establish an always-enabled negotiated mode.

Receive diagnostics identify frame/slice/packet indices, frame-end, slice-end,
packet-enable and last-packet flags. Some paths reject data older than consumed
indices. Other messages cover GDR recovery, I-frame integrity, frame CRC and
received/expected lengths. Some code also serves LTE/SDR fusion, so each mechanism
must retain its path-specific scope. Feedback routes name bandwidth/error state
without proving the physical reverse RF route.

`EVENT_SLICE_AVAILABLE`, `HEVC_EVENT_SLICE_AVAILABLE` and
`wl_codec_append_data_low_latency` show available incremental producer mechanisms.
They do not establish active O4 multi-slice encoding, resumable receiver decoding
or scan-out before picture completion. Compare the separate
[AR9311 incomplete-picture experiments](../soc/ar9311.md) before transferring
such a claim between platforms.

Timing messages warn about decoder submission above 3 ms, decoder-stream delay
above 7 ms, and a display branch ignoring a delay above 20,000 µs. These are
diagnostic/branch thresholds, not measured stage budgets or a 20 ms latency guarantee.

## A configuration that must retain its consumer

The ZV902 periodic-GDR encoder configuration is referenced by **goggles Wi-Fi
sharing**. Its values include:

| Setting | Value |
| --- | --- |
| Codec mode | `DUSS_WL_VENC_MODE_GDR_LIVEVIEW` |
| GDR mode | `DUSSS_WL_VENC_GDR_MODE_QUICK_RECOVER` |
| Software bandwidth feedback | 1 |
| Multi-slice encoding | 0 |
| `two_pass` | 1 |
| Automatic FPS | 1 |
| Bitrate bounds | 512,000–18,000,000 bit/s |

These values are not the O4 air encoder configuration. `two_pass` has not been
semantically mapped and does not prove offline encoding with future-frame
lookahead. A shared-library feature list is likewise insufficient to prove an
active pipeline. The useful design questions are when data becomes available,
how old queued work can become, who retains buffers and when display accepts a
picture—not simply the codec or transport's name.
