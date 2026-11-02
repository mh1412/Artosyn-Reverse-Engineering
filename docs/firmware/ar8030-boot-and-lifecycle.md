# AR8030 boot, worker reconstruction and host lifecycle

[Firmware](README.md) · [Open8030 runtime](open8030-runtime.md)

## Scope and evidence

This consolidates the early Gnd485 18.21.10 receiver reconstruction, including
the hidden historical records behind trials 135–142. Later Open8030 behavior
is documented separately in the [release validation](../experiments/open8030-validation.md).
An early note saying RF was unsupported describes its own reconstruction stage,
not the later runtime's capability.

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

## Source bootstrap and prepared state

The source bootstrap targets `0x0a098a00` and stays below ROM scratch at
`0x0a098ff8`. It reconstructs the observed memory attributes, register handshake,
scratch adjustment and reset-vector contract, omits UART debug output and
introduces bounded waits. Its 792 bytes have SHA-256
`c69e46c57bbb3e5fc7fe0c5581e59fb5735dcf8a97bf0e05ae7aaa0273657768`.
The original 944-byte bootstrap is
`a9e59b0e05c42042e27c35ca998c83326aa9d8ab0cea294bc2d0f7199929df68`.

Fifteen reported comparison cases matched ordered register/scratch stores under
synthetic ready signals and stable timers. UART and ordinary stack/RAM writes
were excluded. This does not prove timing or timeout-path equivalence. Omitting
the bootstrap in trial 137 left SDIO at ROM identity `4152:8030`; trial 138 with
the source bootstrap reached runtime `4152:8031` and the original video stack.

Firmware-owned initialization validates the profile before touching hardware,
probes the matched chip profile, applies A/B pins and two defaults batches,
calibrates, initializes PSRAM and prepares operation 76 with RX/TX held. A
failure stops progression and remains diagnostic state. Repeating the one-shot
initialization is rejected rather than resetting partly active hardware.
The original backend's candidate command starts the prepared receiver.

Boot diagnostic alias `0x015a0024` carries eight little-endian 32-bit words:
version, stage, signed error, default-data byte count, RF bound in milliseconds,
configuration bytes consumed, CRC32C and identity source. In the earliest
compiled-profile stage the last three words were reserved zeros. **Stage 8 means
prepared**, not linked, receiving video or complete firmware compatibility.

## Loader JSON overlay and identity

The configuration reservation is `0x200000`–`0x203fff`. The loader writes the
embedded configuration, the main payload, then the external configuration over
the same reservation. Its external transfer does not append a NUL terminator.

| Matched input | Verified extent |
| --- | ---: |
| Embedded reservation | 16,384 bytes |
| Embedded text before its NUL | 12,052 bytes |
| External file | 10,341 bytes |
| External complete JSON object | 10,340 bytes |
| Remaining old text after the external transfer | 1,711 bytes |

Trial 140 reached native code but failed configuration with −22 before RF.
The saved overlay reproduction, repeated independently during this review,
rejects the entire resulting text as one JSON document while accepting the
external object's exact prefix. Trial 140 did not export RAM bytes; the stale
tail diagnosis is a reproduction from matched inputs, corroborated by the next
trial's consumed-length/CRC, not a direct memory dump of the failed boot.

The loader-specific parser accepts optional BOM/leading whitespace and one
complete root object within the 16 KiB bound. It stops at the matching closing
brace without treating braces inside strings or nested values as the end. It
still rejects malformed/truncated object content and duplicate matching setting
names, including escaped equivalents. Ordinary JSON parsing remains strict over
its complete supplied span; the special boundary applies to loader memory.

Consumed bytes include leading BOM/whitespace, exclude bytes after the closing
brace, and produce CRC32C with **initial zero and no final XOR**. The matched
object gives decimal **1,625,207,254**, independently reproduced. Trial 141 reports
`(1, 8, 0, 5632, 0, 10340, 1625207254, 2)` in the eight boot words.

The recorded OTP identity formula is `(word << 6) | (byte & 63)`, using byte 14
and the little-endian word at bytes 26–29, in the four-byte identity width.
Blank identity permits the JSON fallback; an OTP read error stops initialization.
Identity source values are compiled profile 0, JSON fallback 1 and OTP 2.
Static profile generation now supplies defaults, pins and FCH keys, while JSON,
OTP and the Linux candidate list supply dynamic settings. Its manifest's retained
plan input records provenance; it does not mean every plan setting remains
compiled into the loader-driven runtime.

The [configuration study](ar8030-configuration.md) gives the accepted historical
mode, channel, MCS and calibration subset. Later factory/user precedence and
AP/P1 work must not be inferred from trial 141's matched DEV acceptance alone.

## Continuous clocks and lifecycle

The 40 MHz counter's low word wraps every **107.3741824 seconds**. Removing a
60-second session deadline without changing elapsed-time arithmetic would leave
long-running control vulnerable to that wrap. The continuous path reads a
coherent high/low/high pair and subtracts a 64-bit start value. MCS and FCH
deadlines use that elapsed value; diagnostic microseconds retain their 32-bit
ABI and still wrap after 2³² microseconds.

Short hardware waits use modular low-word differences, while millisecond
deadlines use modular 32-bit comparisons. The first-16-seconds histogram uses
full elapsed time so a later diagnostic wrap cannot repopulate startup buckets.
Synthetic tests covered timer-read rollover, multiple low-word wraps, diagnostic
wrap, millisecond deadline wrap and more than one million polls. The later
hardware observation crossed the low-word wrap, not all those synthetic cases.

Zero duration selects continuous preparation only through the boot API. The
bounded diagnostic preparation API still rejects zero. Explicit stop holds
RX/TX and cancels pending synchronization; restart requires preparation again.

| Host command | Recovered lifecycle operation |
| --- | --- |
| `0x0b000000` | Start from initialized/stopped state |
| `0x0b000001` | Stop from running state |
| `0x0b000002` | Initialize |
| `0x0b000003` | Deinitialize from stopped state |

These use zero-length payloads. Native lifecycle preserves configuration and
candidates, keeps socket/copy queues allocated across RF pauses, and prevents
candidate updates from restarting an explicitly stopped receiver. Stop waits
with a bound for pending RX DMA; timeout retains storage ownership and blocks
restart. Static native init/deinit deliberately differs from the original RTOS
allocation and rejects duplicate init.

Trial 142 exercised two restart sequences, but needed no RX/TX cancellation.
Its stuck-DMA and late-completion paths therefore retain synthetic coverage.
The [early receiver trials](../experiments/open8030-early-dev.md) give the actual
video observations, failures and recovery boundaries.
