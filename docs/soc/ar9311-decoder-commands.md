# AR9311 stock decoder command dispatch

[SoC studies](README.md) · [Slice-decoding experiments](../experiments/ar9311-slice-decoding.md)

The complete `VPU_DecGiveCommand` implementation in the retained Artosyn
`libmpp_service.so` distinguishes accepted configuration, cached metadata and
backend operations. A successful return does not generally establish fresh
decoded output or successful hardware action. Its command numbers also differ
from the related SDK header used for comparison.

## Binary identity and command numbering

The compared 17.5.3 and 17.5.8 libraries are byte-identical, 4,394,048 bytes,
with SHA-256
`e480bdc292ede1ca19e24998c7323fbc0218fffe1f8f5d569f1fae384d2f715f`.
The function occupies virtual addresses `0xe17a0` through `0xe2d38`
(exclusive): 1,382 AArch64 instructions. Both disassembly listings match the
ELF bytes and symbol extent; their only textual difference is the filename
header. This is one implementation retained under two firmware labels.

After instance and product-capability checks, an unsigned comparison admits
command IDs 0–97. The 98 signed 16-bit entries at `0x2a92a8` select targets as
`0xe183c + 4 * sign_extend(entry)`. There are 58 distinct destinations, including
the default rejection path. IDs 51–63, 66, 68, 70 and 72–96 all return 4;
out-of-range IDs take the same path. A failed earlier capability check can
return before command dispatch.

The related StarFive SDK snapshot at revision
`b60da16be1b36453aa46599889c28310fd148fb8` provides useful names and comparison
code, but its `wave420l` command enumeration is not this binary's ABI:

| Operation identified from behavior | Artosyn binary ID | Related SDK enum ordinal |
| --- | ---: | ---: |
| Set low-delay configuration | 25 | 29 |
| Get low-delay output | 26 | 30 |
| Get cached queue status | 67 | 70 |

Four CU-report entries precede low-delay configuration in that header. Later
differences mean even a uniform ordinal adjustment is insufficient. In the
Artosyn binary, command 30 copies a 744-byte indexed record into the supplied
argument; it does not request low-delay output. These numbers describe the
inspected binary. A matching public header and complete parameter layouts have
not been recovered.

## Dispatch map

Names below describe observed behavior. They do not assert that every name or
structure matches an original Artosyn declaration. `state` means the pointer
loaded from instance offset `+32`; offsets are bytes.

| IDs | Observed operation and boundary |
| --- | --- |
| 0–7 | Enable/disable rotation or mirroring; set direction, angle, output descriptor and stride. Enable requires an existing stride. Angle accepts 0/90/180/270; stride must be nonzero, eight-aligned and sufficient for the relevant dimension. |
| 8 | Copy 312 bytes of saved sequence information from `state+104`. |
| 9–10 | For codec mode 0, tail-call `SetParaSet` with selector 0 or 1 and retain its return. |
| 11 | Store a word at `state+0x9ecc`, restricted to product values 2–4. |
| 12–13 | Enable/disable deringing; enabling requires a stride. |
| 14 | Save secondary-AXI configuration with different word ordering for legacy and Wave product families, plus codec/product restrictions. |
| 15–16 | Set/get a 24-byte saved record at `state+0x9eb0`. |
| 17–23 | Enable/disable userdata, set address/size/report mode, attach virtual userdata storage and save a 16-byte configuration record. |
| 24 | Copy 616 bytes from `state+0x9c40` after its availability flag is set; otherwise return 11. |
| 25–26 | Configure the legacy low-delay path or retrieve its metadata; detailed gates below. |
| 27–28 | Store words at `state+0x415c` and `state+0x409c`. |
| 29 | Return 1 after the common preflight checks. |
| 30 | Read an index from the argument and copy its 744-byte saved record back into that argument. |
| 31–32 | Enable/disable saved reordering state, with initialization and codec restrictions on legacy product values 0–1. |
| 33 | For codec mode 0, store a word at `state+0x9eac`. |
| 34–36 | Free selected DMA allocations, export allocation/frame metadata, or clear allocation bookkeeping; these are distinct operations. |
| 37 | Set `state+0x9ec8` to 1, the flag used by the Wave5 submission path to select base option `0x10`. |
| 38 | Set a display flag through a product backend or a saved bitmap, depending on the per-core capability. |
| 39–40 | Read cached scaler information or validate and save scaler configuration. |
| 41–44 | Store words at `state+0x9ee0`, `+0x9ee4`, `+16320` and `+16324`; ID 42 requires codec mode 26. |
| 45–46 | Free indexed FBC Y/C-table allocations or an indexed motion-vector allocation. |
| 47–49 | Calculate and allocate indexed Y-table, C-table or motion-vector storage. |
| 50 | Store a word at `state+0x9edc`. |
| 64–65 | Set/clear the instance logging flag at `+24`. |
| 67 | Copy two cached queue-counter words from `state+0x9ee8`. |
| 69 | Under the core lock, call `ProductVpuGetBandwidth`, retaining the backend return. |
| 71 | Store a word at `state+0x9ef4`. |
| 97 | Under the core lock, call `ProductVpuGetDebugInfo`, retaining the backend return. |

Null and range checks are case-specific. For example, commands 8, 22, 27,
28, 30 and 67 do not perform a local null-argument check. Indexed metadata and
allocation cases also lack a general index-bound check in this function.
The map therefore does not define a validated standalone calling interface.

## Low-delay configuration and output

Command 25 requires a non-null argument and product value 0 at instance
offset `+20`. Other products return 19 even when the requested enable value
is zero. When enabling, codec mode at instance `+12` must be 0 and rotation,
mirroring, tiled-to-linear conversion and deringing must all be disabled;
otherwise it returns 3. Disabling bypasses those mode/transform checks but
retains the product check.

An accepted request copies the enable and row-count words into
`state+0x414c` and `state+0x4150`. The setter does not validate the row-count
range, call a backend or write a hardware register. Related SDK source names
this legacy combination CODA980/AVC; it does not establish a Wave521C control.

Command 26 requires a non-null argument, enabled low-delay state, codec mode
0 and the same instance returned by `GetPendingInst`. Disabled or wrong-codec
requests return 4; a different pending instance returns 11. It then calls
`GetLowDelayOutput` and discards that helper's return, using the dispatcher's
existing success value of 0.

The [complete helper review](../experiments/ar9311-slice-decoding.md#complete-low-delay-helper-and-wave5-submission-review)
shows register reads and conditional metadata assembly, with a constant helper
return of 1. Neither that return nor the dispatcher's 0 proves newly available
pixels, valid output indices, or resumable partial-picture decoding.

## Cached status and scaler behavior

Command 67 copies eight bytes already present in decoder state. It does not
query the VPU, wait for a command or establish output completion. The
[Wave5 submission review](../experiments/ar9311-slice-decoding.md#complete-low-delay-helper-and-wave5-submission-review)
separately identifies the register read that updates cached queue counts during
submission. A cached count's meaning and freshness depend on that update path.

Commands 8, 24, 30, 35 and 39 likewise export stored information or allocation
descriptors. Copying those records does not copy newly decoded pixel planes or
establish a coherent hardware snapshot.

Command 40 requires sequence initialization and a non-null argument. If its
enable word is exactly 1, it validates the requested dimensions against
sequence dimensions and capability-dependent alignment, then saves enabled
scaler state and dimensions. Any other enable value returns success without
clearing the previous scaler state. Consequently, a successful request with
enable 0 does not demonstrate that a previously enabled scaler was disabled.

## Display flag and return handling

For command 38, a per-core queue-support capability equal to 1 selects the
locked product-backend path. After acquiring the lock, the function calls
`ProductVpuDecSetDispFlag`, discards its return, releases the lock and returns
the existing success value. If lock acquisition fails, it returns 1 rather
than forwarding that lock result. The bandwidth/debug getters retain backend
results, so failure masking is not uniform across commands.

The other display-flag path acquires a display lock, ORs a bit into the saved
word at `state+756`, and releases that lock. It does not check the lock return
or validate the supplied bit index. The AArch64 32-bit variable shift uses the
low five index bits: 32 selects bit 0 and -1 selects bit 31. This describes the
binary instruction's behavior, not a valid index domain for callers.

## Allocation ownership and reset boundaries

Command 34 calls DMA-free helpers for eligible records, including three
62-entry arrays and additional allocations subject to external-ownership
flags. It ignores free-helper returns and contains no hardware-quiescence
check. Command 36 clears saved allocation bookkeeping and flags without
calling a free helper. Clearing state therefore does not establish that the
associated memory was released or that device DMA stopped.

Commands 45–49 operate on indexed FBC-table or motion-vector records without
a local bounds check. Allocation paths update record metadata before calling
the allocator and translate a negative allocator result to return 17. The
dispatcher does not establish rollback or recovery after those failures.

Similarly, command 20 checks that userdata base/size and the supplied virtual
pointer exist, then writes the DMA record before attempting attachment. An
attachment failure returns 17 without restoring the prior record. Callers
must account for state mutation on an unsuccessful request; a return code
alone does not describe the remaining ownership state.

## Verification limits

Offline checks matched all instruction words to the ELF, decoded the complete
98-entry jump table, traversed branches within the function and compared the
selected SDK enum ordinals. They also anchored the ignored display-backend
return and variable-shift instruction. The duplicate listing was compared in
full.

This review does not execute the function, audit every called backend, recover
all C types or test hardware behavior. The related SDK header and implementation
were consulted in selected sections only. The full library, framebuffer setup
and sequence-initialization review remain open; no new slice-ready or latency
result follows from completing this dispatcher.
