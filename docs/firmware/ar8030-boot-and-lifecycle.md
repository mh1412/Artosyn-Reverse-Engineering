# AR8030 boot, worker reconstruction and host lifecycle

[Firmware](README.md) · [AR8030](../soc/ar8030.md)

## Scope and evidence

This study reconstructs the stock Gnd485 18.21.10 radio image.

The matched stock container is SHA-256
`dc58247910af564376350404383c5c411cbec21663873fba24e70c61a3584e3c`;
its main payload is
`3e454131a334ec422c0b2e8ae4924d6bda68cb61b70fd754da26412578b39c87`.
The main starts at container offset `0x4600`, loads at `0x204000` and has
**412,736 declared bytes**, followed by 448 container-padding bytes. Hashing the
whole remaining container is not equivalent to hashing that payload.

The documentation review independently checked those identities, decoded the
five task descriptors and reproduced the configuration-overlay arithmetic.
Other instruction semantics below are retained findings from the original
analysis, not a new execution of the radio binary.

## Recovering worker boundaries

Generic disassembly had treated compiler save/restore helpers as non-returning,
truncating callers. The reported instruction trace identifies `jal t0,0x24dbc2`
as a save helper returning through `t0`; the restore tail at `0x24dbea` returns
through `ra`. The division-shaped helper at `0x24daea` uses that tail, and
`0x21a860` reads the timer and divides by 40. Incorrect fallthrough recovery made
ordinary control flow look like dead code or a failed start path.

The analysis annotations repaired selected CALL/return flows without modifying
firmware bytes. They did not establish every C prototype, ABI or custom opcode.
The common worker entry `0x21d2e8`, with its own 304-byte stack prologue after a
trap sequence, also had to be split from the preceding debug worker.

Five descriptors begin at `0x267270`, each **44 bytes** long. Offset 0 is task
ID, +4 callback, +8 name pointer, +12 raw priority, and the low/high halfwords
at +16 hold raw stack/queue values. The initial decoder had confused the ID word
with the callback; the corrected layout agrees with the actual image bytes.

| Task | Descriptor | Callback | Raw priority | Raw stack halfword | Queue capacity |
| --- | --- | --- | ---: | ---: | ---: |
| PHY | `0x267270` | `0x225d82` | 38 | 6144 | 20 |
| Link | `0x26729c` | `0x22de32` | 39 | 5120 | 10 |
| MAC TX | `0x2672c8` | `0x237d28` | 48 | 4096 | 5 |
| MAC RX | `0x2672f4` | `0x232eca` | 40 | 6144 | 10 |
| Debug | `0x267320` | `0x21d068` | 21 | 5120 | 10 |

The stored labels are `bb_phy_task`, `bb_link_task`, `bb_mac_tx_task`,
`bb_mac_rx_task` and `bb_dbg_task`. Raw priority/stack values cannot be translated
directly into Zephyr scheduling policy without the original OS conventions.

The recovered initialization lead at `0x242cf2` registers commands, creates
five queues with 256-byte messages and their workers, broadcasts message 0,
then stores state 1 at `0x26d173`. Start at `0x242be0` requires state 1 or returns
−7, broadcasts message 1, waits for all five descriptor state words at +`0x20`
to reach 1, then stores global state 2. The common worker receives via the queue
at descriptor +`0x28`, invokes the callback with state/message/payload/length,
and records processing time. Observed message 8 leads to state 4 and suspension;
that is not a complete worker-message enumeration.

PHY state-0/message-0 reaches calibration through `0x22373a`; message 1 begins
search/start behavior. The register-field RMW helper at `0x2057ba` uses base
`0x0a110000`, an encoded byte offset, shift and width/full-word rule. An earlier
experimental UART interpretation of that base was not established.

The partial instruction model added the observed even-register RV32 forms of
packed `SUB64` (`0xc2001077`) and unsigned `MULR64` (`0xf0001077`), alongside
ADD64. Synthetic overlap, borrow and high-product cases support those modeled
forms, not a complete CPU identification. Later callback exports removed the
remaining generic custom sites; zero unresolved sites still does not prove
register, scheduler or lifetime semantics.

## Loader configuration and host lifecycle

Configuration occupies `0x200000–0x203fff`. The observed loader writes embedded
configuration first, then overlays an external JSON file without appending a
NUL terminator. In the matched input, the embedded text was 12,052 bytes and
the external file 10,341 bytes; 1,711 bytes of old text can remain after overlay.
A reader must distinguish the complete root object from stale reservation bytes.

The recorded OTP identity formula is `(word << 6) | (byte & 63)`, using byte 14
and the little-endian word at bytes 26–29 in a four-byte identity width.

| Host command | Recovered lifecycle operation |
| --- | --- |
| `0x0b000000` | Start from initialized/stopped state |
| `0x0b000001` | Stop from running state |
| `0x0b000002` | Initialize |
| `0x0b000003` | Deinitialize from stopped state |

These recovered lifecycle requests have zero-length payloads. Their existence
does not establish the complete stock RTOS allocation or failure behavior.
