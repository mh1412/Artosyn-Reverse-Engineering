# AR8030 receive acquisition: power activity to valid MAC control

[Experiments](README.md) · [Startup reconstruction](../soc/ar8030-receive-startup.md)
· [Later receiver integration](open8030-early-dev.md)

Historical scope: Gnd485 18.21.10 DEV receiver, trials 60–96, with the first
native MAC/parser results recorded on 14 October 2026 UTC. These experiments
progressed from changing power samples to two DMA completions and then one
control packet parsed on the native radio. They did not yet establish a linked
native receiver or native video. Those later results are covered separately.

The main question was why an executing native runtime could configure the
radio and observe interrupts without receiving usable packets. All register
offsets below are relative to `0x0a110000`. Diagnostic operation numbers name
experimental branches, not host protocol commands.

## Setup and observation limits

Most tests used 5,770,000 kHz and bounded RAM-loaded firmware retaining the
stock bootstrap/configuration. Early operations could inherit peripheral state
within one load; later comparisons used separate loads and stock restoration.
The usual early window was only 1–1.5 seconds. Source-7 command interrupts,
changing power samples, source-2 attribution, hardware-valid headers, DMA
completion and checked MAC contents were separate counters.

Recorded restoration checks returned to stock firmware, rebooted the receiver,
checked 24 getters and captured approximately 301 nonblack frames through a
1920×1080 YUYV/30 fps capture device. Several runs retained four warmup errors
outside the measured interval. These checks support stock recovery. Capture-card
cadence is not native RF reception, over-air loss, or glass-to-glass latency.
No hardware or image-viewing check was repeated for this documentation review.

## Startup changes produced activity without packets

| Trial | Change or control | Observation |
| --- | --- | --- |
| 60 | Operation 36: shared identity clear and single-channel retries | 64 restarts in 1,000,004 µs; no valid headers or DMA |
| 61 | Operation 37: preserve TX hold during search | 64 restarts in 1,000,012 µs; intended `800` bit changes, still no headers/DMA |
| 62 | Operation 38: power-only initial-search commands | 2,857 power samples in about one second; no receive DMA |
| 63 | Operation 39: 42-channel receive search | Every channel visited, 101 restarts and two wraps in 1.5 seconds; no reception |
| 64 | Operation 40: 42-channel power diagnostic | 4,173 samples, 149 restarts and three wraps; no reception |
| 65 | Operation 41: configured 5 MHz RX bandwidth only | Readback `20b0c2ad`; no headers/DMA |
| 66 | Operation 42: configured interleaver and duration too | Readback `20b0a2ac`, 7,096 µs frame; no headers/DMA |
| 67 | Instrumented stock skips the receive-start helper | Stable prestart snapshots; observer success is not native reception |
| 68–69 | Operation 43: startup upper byte `3c`, then source-2 attribution | 258 source-2 events in each run; no headers/DMA |
| 70 | Missing plan during staging | No radio load ran; not a hardware failure |
| 71 | Operation 44: restore selected compensation defaults | 257 source-2 events; no headers/DMA |
| 72 | Same trial-71 image, 5,839,000 kHz | No source-2 events; no headers/DMA |
| 73 | Operation 45: restore DEV `B4C` field | Readback 8 and 257 source-2 events; no headers/DMA |

Trial 62's raw power values ranged from 42,619 to 540,327. Trial 64's strongest
channel was 5,770 MHz: maximum 591,947 across 84 samples/three visits, while
other channel maxima were below 120,000. These are sequential, uncalibrated
measurements. They do not establish peer identity, RSSI units or VTX hopping.
Matching stock/native `A30 = 162110` and `AFC = 40020008` likewise did not
prove the complete analog path.

In trials 69/71, source 2 sampled `E6C = 00180000`, attributed to raw user 8,
mode 1, with both SNR fields zero. A frequency-dependent interrupt is useful
evidence of activity, but is not a decoded header. The implementation continued
searching instead of treating source 2 as packet acceptance.

## Hardware validity did not validate the extracted contents

Operation 46 combined startup upper byte `3c` with the earlier 20 MHz,
length-24 interleaver hypothesis (`9E0 = 3cb0c4ad`), acquisition identity zero,
TX held, default compensation and the restored `B4C` field. Trial 74 observed
159 header-source interrupts and one hardware-valid indication, but no DMA.
The diagnostic stopped search at that indication. Trials 79–82 repeated one
valid indication per run; added capture then showed seven zero control bytes.

The [FCH checksum](../protocols/ar8030-wireless-format.md#seven-byte-phy-control)
computed `0x1e` for those zero bytes, disagreeing with stored zero. Later
end-of-packet snapshots did not fix the extraction. Trial 82 also observed zero
`F00` and control words before configuration and zero `F00` while RX remained
reset, excluding the simple explanation of a stale preconfiguration `F00`.

| Trial | Hypothesis tested | Result |
| --- | --- | --- |
| 78 | Operation 47: linked identity `cb` in both identity fields | No valid headers/DMA; not an acquisition fix |
| 83 | Trial-82 image at 5,839 MHz | No activity in the bounded comparison |
| 84 | Operation 48: upper byte `20`, acquisition identity zero | No valid headers/DMA; not a complete linked-state reconstruction |
| 85 | Operation 49: clear `A48` for the native TX bandwidth codes | One valid indication, zero extracted control, no DMA |
| 86 | Verify bank selector and sample again after 1 µs | Selector held; bank values differed but delayed values did not; no DMA |
| 87 | Operation 50: force one user-8/MCS-1 buffer transition | `BF0` changed `41410100 → 41410101`; `D08` bit 16 was already clear; no DMA |
| 88 | Operation 51: resume search only for all-zero controls | Four valid indications; fourth control was nonzero but still checksum-invalid; no DMA |
| 89 | Operation 52: resume search for every checksum failure | 97 valid indications, all 97 failing the check; no DMA |

Trial 88's nonzero control was `00000000000004`: computed `0x0e`, stored zero.
Trial 89 ran 1,500,004 µs with 96 restarts and 153 header interrupts. Its failed
controls comprised 94 empty and three nonempty results. It removed the earlier
diagnostic latch artifact, not a radio-imposed one-header limit. Aggregate
classification relies on the checked implementation; individual raw snapshots
of all 97 samples were not retained.

The original lookup table contains the recovered XOR-basis mapping for indices
0–247 and zeros for 248–255. The historical C implementation was compared with
all table entries and 10,258 vectors. During this review, both full original
checksum tables were checked independently again. The eight-bit control lookup
must not be replaced by a conventional CRC-8 solely because a report calls it
“CRC8.”

Prefixes such as `10`, `32`, `54`, `ba`, `dc` and `fe` in `F00` resembled a test
pattern, but also occurred in working-stock observations. That resemblance
does not establish native PSRAM contamination, packet identity or valid control.
The forced buffer transition of trial 87 likewise was not a complete stock MCS
policy, peer transition or MAC transmitter setup.

## Packaging failure interrupted the receiver comparisons

Trials 75/76 failed to produce native HELLO and remained at ROM SDIO identity
`4152:8030`. The executable was 15,300 bytes, declared as 15,304 after normal
eight-byte padding. A complete receiver reboot did not resolve trial 76.
Trial 77 moved/aligned a reset hook and changed layout, but still failed;
that edit was reverted. An earlier instruction interpretation was also corrected
using the matching ELF/disassembly. None of this proved an ISA alignment rule.

Trial 78 used the original executable with a 15,360-byte declared extent,
rounded to 512 bytes, and booted. The retained 33,280-byte containers are
**identical after their 64-byte headers**. Only header offsets 8, 9, 28 and 29
changed: the header checksum and declared runtime length. Their additive runtime
checksum was identical because the extra bytes were zero.

| Candidate | Declared runtime | Container SHA-256 |
| --- | ---: | --- |
| Failed trial 75 / repeated 76 | 15,304 bytes | `5cc32d606b24a9615dd8c2be778f539ff40f51bf48b901f12ba6757a00abf874` |
| Booted trial 78 | 15,360 bytes | `210ba0d0286f98d4e3b249bd1800dc60a256a1e248d29faaff8f14ed22fe10a2` |

The native harness adopted 512-byte padding as a measured workaround; the
general packer retained an eight-byte default and instrumented stock used that
default. Earlier non-sector-sized payloads had booted. The exact loader
constraint remains unresolved. Pre-run manifests marked boot acceptance false;
the later trial receipt supplies the successful observation.

## Establishing a working isolated-stock reference

After trial 89, trials 90–94 asked whether the isolated RAM-trial environment
could acquire with stock firmware. An observer redirected GET_REG reads from
baseband into RAM and sampled a seven-byte cache at `0x2684b9`. Stock fills
that cache only after control checks and other gates. Reads were unsynchronized
and could retain or tear data; the cache did not record every hardware header.

| Trial | Stock setup | Observation |
| --- | --- | --- |
| 90 | Instrumented observer, about four seconds | Cache stayed zero |
| 91 | Same observer, about twelve seconds | Cache zero; enabled slot 0 stayed state 0/MCS 25 |
| 92 | Observer plus stored candidate-list command | State 2/MCS 1 by sample 40, MCS 7 by sample 59; cached control passed check |
| 93 | Observer, omit candidate command, about seventeen seconds | Cache zero; all six status snapshots stayed state 0/MCS 25 |
| 94 | Byte-identical original stock image plus candidate command | State 2/MCS 1 at 8.05 seconds; MCS 7 at 11.86 seconds |

The normal Linux backend sends `0x02000004` with a 402-byte slot/count/address
list, allowing up to 100 four-byte candidates. The experiment sent slot 0 and
the two stored nonzero candidates. The radio handler updates its in-memory
list; this command does not itself persist configuration. The comparison
supports a missing host-configuration dependency in the earlier isolated tests.
It does not prove that all acquisition timing differences came from that setter.

GET_STATUS is 300 bytes. Linux backend DWARF places the six-byte link entries
at offset 252, state at entry byte 0 and MCS at byte 1. Only enabled slot 0 was
interpreted; unused entries contained uninterpretable bytes. Linux SDK types
are not original radio declarations, and the returned configured peer field
is not independent evidence identifying the physical transmitter.

Trial 92's cached `0000000000000c` computed checksum zero, matching its stored
zero. Trial 94 kept RX8 at configured **5 MHz**, `9E0 = 3cb0a2ac`, and duration
`970 = 12501250` through acquisition. Linked snapshots released TX hold,
changed `A48` from `80808080` to zero, changed `BF0` to `41410101`, and updated
the identity byte from zero to `cb`; `D08` stayed zero. Sequential register
snapshots were not atomic initialization recipes. The normal backend's later
20 MHz profile was a separate state.

## Trials 95 and 96: independently valid control packets

Operation 53 returned to operation 45's configured 5 MHz startup, retained the
pre-TX `A48 = 80808080`, added checksum-aware retries and extended the window
to ten seconds. It did not force the trial-87 buffer transition, send the vendor
candidate setter, or implement native peer acceptance.

| Measurement | Trial 95 | Trial 96 |
| --- | ---: | ---: |
| Frequency | 5,770,000 kHz | 5,770,000 kHz |
| Elapsed monitor time | 10,000,030 µs | 10,000,026 µs |
| Search restarts | 562 | 561 |
| Hardware-valid indications | 2 | 2 |
| DMA completions / packet count | 2 / 2 | 1 / 1 |
| Total bytes | 96 | 48 |
| Invalid DMA packets | 0 | 0 |
| First relevant time | Header at 7,062,367 µs | Packet at 7,055,905 µs |
| Native MAC / full-control valid counts | Parsers not yet wired | 1 / 1 |

The first observed activity arrived after roughly seven seconds, well beyond
the earlier 1.5-second windows. Duration and startup profile changed together;
this does not isolate a single universally necessary register correction.

Trial 95 retained only the **last** completed packet: a field named `first_bytes`
actually held the latest prefix. Its 48 bytes fit entirely in that 64-byte
diagnostic area. The first packet was overwritten and cannot be independently
validated from this retained prefix. The unpopulated `last_mcs` field was not
a measured MCS value.

Each retained packet has an eight-byte MAC header, zero payload descriptors,
39 bytes of full control and one alignment byte. The padding byte is outside
the header and declared-control checks. Neither packet contains video data.

| Retained packet | Header CRC16 | Full-control check | SHA-256 |
| --- | --- | --- | --- |
| Trial 95 last packet | `e29f` | `26` | `f5f96d44d3d42b3ef28bec500cf70a22287602138110b92a9c279be61b012b2d` |
| Trial 96 packet | `a04c` | `1e` | `9a3a9ab6e2e2af1bc844763fb970510430964e2da6905ff87abd32c7593d9bd2` |

Original table `0x2533e8` matches all 256 entries of non-reflected CRC16
polynomial `0x1021`, initial zero. It checks the MAC header/descriptors with
bytes 2–3 zeroed. The separate lookup table at `0x253668` checks declared full
control with byte 2 zeroed, **without** the seven-byte FCH nibble normalization.
The firmware/container identity and both table mappings were rechecked, and
both retained packets' checks and extents reproduced during this review.
Packet identities and address bytes need not be published to explain the format.

Both trial-95 FCH-register checks still failed while the DMA packet's independent
checks passed. Therefore that particular FCH diagnostic cannot serve as a
universal gate rejecting MAC packets. Its relationship to the received MAC
data remained an open question at this stage.

## Native parsing was still separate from peer admission

Trial 96 added bounded MAC and full-control parsing synchronously at DMA
completion. Parsers borrowed the DMA buffer; validated control was copied before
advancing the ring. The latest-prefix field was cleared before copying, preventing
stale tails after short packets. Unsupported compact/extension formats were
distinguished from corrupt packets. The single control had extension bitmap
12: address-list and slot/MCS-list views, with channel indices 24, corresponding
to 5,770 MHz in the recorded plan.

An additional side-effect-free idle admission routine was compiled but not called
by this receiver. Linker garbage collection left the trial-96 binary unchanged.
Host replay of both retained packets returned READY for slot 0, peer XOR `cb`
and channel 24; removing the configured candidate list rejected the peer.
This was an offline decision, not a native link transition.

Admission checks included receive-channel/relay agreement, pairing suppression,
local identity at position zero in the peer's address extension, candidate or
configured-peer match, mode policy, channel bounds and free-slot fields. The
peer's address extension is distinct from the receiver's stored candidates.
Mode changes required applying policy and rebuilding the frame before reevaluation.

Stock's accepted transition then maps logical RX user 8 to physical buffer user
0, writes **1** to `BF0` bits 0–1, cancels search, retunes, queues MAC task message
`0x302`, installs peer identity and releases the held transmit engine. The MAC
TX callback creates a context with packet-ring/queue ownership. State 2 needs
later login/token/synchronization checks. A parser READY result, or copying the
visible reset bits, implements none of those coupled ownership requirements.

Historical host tests covered 10,000 MAC messages, 10,000 full controls and
20,000 checksum corruptions against the original tables, plus extent, extension,
immutability and admission cases. These historical sanitizer/build results are
distinct from the two-packet read-only verification repeated here. Sustained RF,
peer transitions, payload reassembly and video were later work; see the
[wire formats](../protocols/ar8030-wireless-format.md),
[link state study](../protocols/ar8030-link-management.md) and
[later validation](open8030-validation.md).

The next historical stage is [first uplink and AP login](ar8030-uplink-and-login.md):
three local TX completions, a corrected startup profile, a stock report reference,
matching AP assignments and bounded request-ID recovery.
