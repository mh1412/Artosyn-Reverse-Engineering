# Wave521C codec firmware reconstruction

[Firmware](README.md) · [Slice experiments](../experiments/ar9311-slice-decoding.md)

AR9311's ARM host executes Linux and the Wave5 driver. The codec has a separate
VCPU firmware image and hardware decoding engines. The October 2026 Gnd485
investigation recovered selected firmware encodings, localized an input wait
and validated a two-byte diagnostic patch. It did not identify a complete ISA,
toolchain, named RTOS or source-buildable firmware replacement.

## Matched firmware identities

| Image | Bytes | Reported revision | Recorded AR9311 result |
| --- | ---: | ---: | --- |
| Artosyn `chagall.bin` from Ascent 17.5.8 | 998,144 | 232676 | Exact tested H.264/HEVC decode |
| TI Wave521C comparison image | 999,104 | 363254 | Responded during initialization; both decode tests failed |

SHA-256 identities:

```text
Artosyn  99bc74cdf1e7ded785c2b65440c983f4e128727eb2bbdf309b14ddde668fd8ac
TI       e77e73a5379b93ba5ae208ff91480f832a89d47eaf1ec42c88f6bcf86cff1fd8
```

The comparison image was taken from linux-firmware revision
`bf992df7a95b5e19bb8f8546eb4d7c53ab2598d0`. Cold testing returned firmware error
`0x2`, STREAMON `EIO` and zero output for both codecs. Device-node registration
was therefore not successful compatibility. The precise cause remained unknown.
The two images shared 402 unique printable strings of at least eight characters;
same-offset byte equality was dominated by zeros and did not establish equivalent
code or board compatibility.

The original receiver research loaded the proprietary blob at runtime and kept
it outside the source tree. Later packaging and other board images have their
own inputs; the P1 encoder firmware must not be substituted on the basis of the
Wave521C name alone.

## Loader and static structure

The examined loader copies the blob unchanged and maps two 1 MiB VCPU code pages
starting at virtual zero. Observed code addresses agree with file offsets. This
does not establish the complete data or stack address space.

The first 1,024 bytes contain 256 little-endian even pointer-like values, with
34 distinct targets. A vector-table interpretation is plausible, but entry
meanings remain unverified. The last nonzero byte is at `0x529e8`, followed by
659,735 zeros. Total zero bytes are 703,200; not all zeros are proven padding.

Strings suggest a C runtime and task scheduler, including seek, scheduler and
stage diagnostics. They do not identify a specific RTOS or prove a bare-metal
design. Older CODA BIT disassembly research has different control-flow encodings
and was used only for comparison, not as a trusted Wave5 decoder.

## Partial encoding recovery

Repeated prefix halfwords appear to extend immediates in 14-bit groups. Combining
them with the low four bits of an `0xa...` operation yielded 501 exact printable
string starts in the scanned region. One-prefix immediates are sign-extended:
runtime registers held `0xffff5018` and `0xffff540c`, superseding the early
unsigned interpretation `0x35018`/`0x3540c`.

For candidate `0xdf..` relative calls, prefix bits and the low eight instruction
bits form a signed displacement, multiplied by two and added to the address
after the instruction. All 5,373 candidates stayed within the scanned code
region, and all 6,144 sampled return addresses matched candidate call returns.

For example, halfwords `7ff8 dfcc` at `0x21de` predict a call to `0x137a` and
return to `0x21e2`, matching runtime observations. This validates selected
relationships, not every instruction interpretation. Branch conditions,
load/store semantics and stack operations remain provisional.

The diagnostic register interface exposes 16 unnamed general-register slots
plus CR0/CR1, ML/MH, LR, PC, SR and SSP. A tentative stack walk failed its
saved-return-address checks. No valid backtrace or complete explanation of the
mapping/cache/prologue mismatch was established.

## Input starvation and parser state

During delayed-slice tests, 4,095 of 4,096 early PC samples fell in
`0x21dc–0x2220` or helper `0x137a–0x1394`; all early LR samples were `0x21e2`.
The outlier `0x35e2` was retained, with interrupt handling only a hypothesis.
Direct host PC/LR registers are offsets `0x0004`/`0x0008`.

Later read-only FIO observations established:

| FIO offset | Observed role |
| --- | --- |
| `0x5008` | DMA start address |
| `0x500c` | DMA end address |
| `0x5018` | SDMA busy status |
| `0x501c` | DMA last address |
| `0x540c` | Parser status |

All 768 samples across two context runs placed DMA last/end addresses at the
exact supplied-byte boundary. Busy status was 0 during withholding and 4 in
sampled completed-picture states; parser status was `0x3000`. Those values do
not independently decode every bit meaning.

Work memory changed as slices arrived and remained stable when input was held
fixed. Global `0xf3cd0` held pointer `0x30bb5000` within that instance's work
allocation at `0x30a00000`, size 1,826,816 bytes. The sampled context field
`+0x3cc` was zero, with unknown semantics. These are instance observations,
not universal physical-address assignments.

Input/parser progress without output-memory changes suggests the wait concerns
input parsing/DMA. Simply bypassing it could expose incomplete parser state;
it is not an identified switch for safe partial-picture reconstruction.

## Validated diagnostic patch

The RAM-only probe changed exactly two bytes at file offset `0x21de`:

```text
before: f8 7f
after:  01 de
```

The inferred short branch skips the helper call to `0x21e2`; bytes `cc df`
at `0x21e0` and the loop's polling conditions remain unchanged. No checksum or
metadata changes were needed. Probe SHA-256:
`48cf3613594033f390b034b4c5976f92476ebca740d7743dc26228f6727ebefd`.

After a cold RAM boot, each codec still produced three exact reference frames.
All 1,024 pre-final LR samples per codec exposed the outer callers:

| Codec | Wrapper | Call into common routine `0x2166` | Observed return |
| --- | --- | --- | --- |
| HEVC | `0x11af0` | `0x11afa` | `0x11afe` |
| H.264 | `0x1d130` | `0x1d13a` | `0x1d13e` |

Captured code memory verified the changed bytes and the old helper disappeared
from withholding samples. No early linear/FBC/table/MV updates appeared. This
is evidence for the diagnostic branch and caller identification, not a
slice-decoding feature patch.

## Picture boundaries and next reconstruction target

The temporary progressive-input driver can refill an active decoder job.
Prematurely marking each packet as picture end can return incorrect completed
frames despite successful dequeue. Without a real picture end, tested streams
waited for following VCL or EOS; AUD/filler alone did not reliably end the wait.
Explicit end after the true final slice removed that extra whole-picture wait
with correct pixels.

In one 720p HEVC timing test, delaying the last slice by one second shifted
reported decode start by approximately one second, while reported decode itself
was about 2.307 ms. The fitted firmware tick rate was approximately 1,171,875/s.
These are firmware timestamps, not independent region-ready or display timings.

The 34 controlled progressive runs summarized in this research produced 102
exact completed frames, without demonstrated early output. Six separate forced
incomplete-picture controls reconstructed supplied bands, but did not establish
continuation of the same unfinished picture. See the
[detailed slice experiments](../experiments/ar9311-slice-decoding.md).

Further RE needs to trace the codec-specific wrappers into picture packaging,
parser state commitment and hardware submission, then establish continuation
and a stable region-completion signal. Asynchronous register samples and copied
memory are observations rather than fences. Full firmware replacement would
also require processor/toolchain and accelerator-interface knowledge beyond
these selected encodings.
