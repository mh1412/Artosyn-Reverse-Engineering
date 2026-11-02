# AR9311 stock decoder initialization and framebuffer setup

[SoC studies](README.md) · [Decoder commands](ar9311-decoder-commands.md) · [Slice experiments](../experiments/ar9311-slice-decoding.md)

The stock Wave5 backend creates a decoder instance, submits sequence
initialization and registers framebuffers in separate operations. These functions
program shared command registers and manage auxiliary DMA allocations. Their
return codes describe the inspected operation; they do not establish completed
picture output or recovery after a timeout.

## Evidence and scope

The review covers all instructions in three retained disassembly listings,
matched against the Artosyn `libmpp_service.so` ELF symbol extents and bytes:

| Function | Virtual address range, end exclusive | Instructions |
| --- | --- | ---: |
| `Wave5VpuBuildUpDecParam` | `0xecb90`–`0xed0c0` | 332 |
| `Wave5VpuDecInitSeq` | `0xed0c0`–`0xed3e8` | 202 |
| `Wave5VpuDecRegisterFramebuffer` | `0xed3e8`–`0xedf10` | 714 |

The 4,394,048-byte library has SHA-256
`e480bdc292ede1ca19e24998c7323fbc0218fffe1f8f5d569f1fae384d2f715f`.
The retained 17.5.3 and 17.5.8 copies are byte-identical. The related StarFive
`wave511` implementation at revision
`b60da16be1b36453aa46599889c28310fd148fb8` helps identify operations, but differs
in feature checks, allocation paths and post-registration setup. Its types and
constants are not a recovered Artosyn ABI.

Here, `state` is the pointer loaded from instance `+32`. Offsets are bytes;
register addresses are offsets in the VPU register window. Numeric codec,
product and map values are retained where the matching declarations are unknown.

## Instance creation

`Wave5VpuBuildUpDecParam` initializes saved register addresses and clears scaler
dimensions before checking the requested format. It accepts these values from
the first word of its parameter block and saves a sequence-change mask at
`state+0x9ecc`:

| Format value | Saved mask |
| ---: | --- |
| 0 | `0x000d0000` |
| 12 or 14 | `0x000d0020` |
| 13 | `0x000f0020` |
| 16 | `0x000d8020` |
| Other | Return 19 |

Thus even a rejected format has already changed some saved state. The related
SDK's capability-bitmap check and external-work-buffer attachment branch are
absent from this function. This does not establish what its callers validate.

For instance product value 4, it sets the work-buffer size to `0x200000`
(2 MiB); values 2–3 select `0x1be000`. Other product values reach allocation
without this function assigning a new work size. It calls the DMA allocator
with allocation type 1. A negative result clears the work record's size and
three address fields and returns 17.

After allocation, the function writes 1 to register `0x194`, obtains common
memory and SRAM descriptors, and saves a temporary-buffer address at common
memory base plus `0x100000`, with size `0x19000`. It does not check the returns
from those descriptor helpers or the memory-clear helper. The clear call uses
the work record's address at `state+16800`; the hardware base register uses
the distinct field at `state+16784`. The helper's address semantics require
separate review.

The main command-register writes are:

| Register | Value source |
| --- | --- |
| `0x114` | Work-buffer address at `state+16784` |
| `0x118` | Work-buffer size at `state+16776` |
| `0x11c` | Saved stream start at `state+776` |
| `0x120` | Saved stream size at `state+816` |
| `0x124` | Low four bits of the inverted endian-helper result, using parameter `+72` |
| `0x13c` | Parameter word `+20` minus 1, without a local nonzero check |
| `0x140` | Parameter word `+100` OR parameter word `+96` shifted left by 2 |

It issues command `0x8` through `Wave5BitIssueCommand` and waits on busy
register `0x70`. A wait result of -1 triggers a work-buffer free and return 16.
There is no preceding stop/reset operation in this function, so this cleanup
does not itself prove device quiescence. A zero success register at `0x108`
also frees the work buffer and returns 1. Free-helper returns are ignored.

On both successful creation and a non-timeout rejection, it reads product
register `0x1044` into `state+0x9ef8`. The values logged from `0x100` and
`0x104` on failure were sampled **before** issuing the create command; they
are not fresh post-failure reason-register reads.

## Sequence-initialization submission

`Wave5VpuDecInitSeq` returns 3 for a null instance or an unsupported bitstream
mode at `state+76`. Mode 0 starts with option 1 only when the escape word at
`state+0x4108` equals 1; mode 2 starts with option 1 unconditionally. A stream-end
word equal to 1 at `state+744` overrides that option to 3. For instance codec
mode 26, `state+92` is shifted left by 2 and ORed into it without a field mask.

The function writes the read and write pointers from `state+736` and `+728`
to `0x118` and `0x11c`, then writes bit 31 OR the bitstream option to `0x120`.
The command option at `0x104` is 1 normally or `0x11` when the saved thumbnail
flag is nonzero. The userdata mask from `state+0x4140` goes to `0x12c`.

It issues command `0x40` and waits on `0x70`. A -1 wait returns 16. Otherwise,
it reads `0x1e0`, saving the low 16 bits at `state+0x9eec` and bits 16–23 at
`state+0x9ee8`, before checking `0x108`. A nonzero success register returns 0.
For a zero success register, exact failure values from `0x10c` map as follows:
1 → 25, `0x40` → 15, `0x20000` → 16, `0x10000` → 28, and other values → 1.
Failure value 1 also causes an additional diagnostic read of `0x110`.

These are command-submission results. This function does not retrieve parsed
sequence dimensions, the required framebuffer count or a decoded picture.
The related SDK has a separate sequence-result path; that separation must not
be erased by interpreting a successful `InitSeq` return as complete decoding.

## Framebuffer resources and allocation formulas

`Wave5VpuDecRegisterFramebuffer` receives an instance, descriptor array, map
value and count. It reads sequence dimensions from `state+104/+108`. Map
values above 16 take the compressed-resource path; map 0 takes the linear
path. There is no general null, descriptor-count or array-capacity check in
this function.

The compressed path allocates motion-vector records at `state+8880`, luma
offset-table records at `state+11360`, and chroma offset-table records at
`state+13840`, each with a 40-byte record stride. It only checks whether each
record's size is nonzero before reusing it. It does not compare an existing
allocation with the newly required size or dimensions.

For ordinary, non-overflowing dimensions, let `A(x,n)` round x up to a multiple
of n, and `M(x) = A(x,4096) + 4096`. The extra page is present even when x is
already page-aligned. Selected formulas verified from the instructions are:

| Resource and branch | Allocation size |
| --- | --- |
| Motion-vector buffer, codec mode 0 | `M(ceil(W/64) * ceil(H/64) * 256 + 64)` |
| Generic luma table, codec modes 0/2/24 | `M(A(W,256) * A(H,64) / 32)` |
| Generic chroma table, codec modes 0/2/24 | `M(A(floor(W/2),256) * A(H,64) / 32)` |
| Product code `0x521d` luma table | `M((A(A(W,16)+16,1024)/16) * (A(A(H,16)+4,32)/4) * 2)` |
| Product code `0x521d` chroma table | `M((A(A(A(W,16)/2,16)+16,1024)/16) * (A(A(H,16)+4,32)/4) * 2)` |

Other codec branches have different formulas. Arithmetic uses 32-bit operations
without overflow checks; these expressions are not acceptance bounds for
arbitrary dimensions. For W=1920, H=1080, the mode-0 MV allocation is 135,168
bytes; generic Y/C tables are 73,728/40,960 bytes, and product `0x521d` Y/C
tables are 77,824/40,960 bytes. These are per-frame auxiliary allocations,
excluding picture planes, work buffers and task storage.

After those arrays, the function attempts a task-buffer allocation without
checking whether a task record already exists. Its size is
`D * state[0x9f00] + F(D) * state[0x9efc]`, where D is the word at `state+20`;
F is D−1 for D=3 or 4, and 2 otherwise. This is again 32-bit arithmetic.
On success it overwrites the task record at `state+0x4200` and writes its
address/size to `0x1d4/0x1d8`. The related SDK uses named buffer-size fields
and build constants here; the numeric expression describes this Artosyn binary.

Any negative allocator result returns 17 immediately. No DMA-free helper is
called anywhere in framebuffer registration, so earlier successful allocations
are retained on failure. Repeated registration, dimension changes and cleanup
must be assessed with the callers and teardown paths. This function alone
provides neither rollback nor proof of a whole-program memory leak.

## Register packing and groups of eight

The first descriptor supplies endian and default stride. Saved index overrides
can select another descriptor's stride without a local bounds check. Descriptor
records advance by 128 bytes. Y/Cb/Cr addresses are loaded from 64-bit fields,
then their low 32 bits are passed as register values; this function does not
reject addresses above the register address width.

For map 0, the initial picture size may be replaced by saved scaler dimensions.
The common picture word includes AXI ID, pixel-order bit 23, output format,
interleave/order settings, stride, and linear/scaler flags. The eight table
entries for saved format values 5–12 encode format fields
`[1,5,2,6,1,5,2,6]` before shifting them left by 20. Format 1 or 9–12 also
sets bit 19. A matching pixel-format enumeration remains necessary to name
these formats reliably.

For a normal positive count N, registration uses `ceil(N/8)` groups. For
group j, the option written to `0x104` combines converted endian shifted by
16, the bandwidth word at `state+88` shifted by 26, bit 3 for the first group
and bit 4 for the last. A single group sets both boundary bits. Register
`0x120` packs the starting index in bits beginning at 8 and the ending index
in the low portion.

Each slot writes Y and Cb at `0x134+16*i` and `0x138+16*i`. Linear map 0
can swap Cb/Cr from the descriptor when the saved order word equals 1.
Uncompressed paths write Cr at `0x13c+16*i` and zero the auxiliary entries;
compressed paths use `0x13c+16*i`, `0x140+16*i` and `0x1b4+4*i` for Y-table,
C-table and motion-vector addresses.

Each group issues command `0x80` and waits on `0x70`; a -1 wait returns 16
immediately. The only success-register read occurs **after the entire group
loop**: zero at `0x108` returns 1, nonzero returns 0. The function does not
check a separate success value after each group, read a failure reason or
roll back earlier groups. Final success therefore does not independently
demonstrate acceptance of every preceding group.

A zero count is not validated as a no-op. On the linear path, descriptor and
register accesses still occur, no group command is issued, and the final
success register is read anyway. The caller's count contract is required to
interpret that result. Unlike the related SDK function, the Artosyn function
also has no final `ConfigSecAXIWave` call.

## Validation limits

Offline verification checked all 1,248 instruction words, function boundaries,
constant tables and call sites. A bounded interpreter evaluated selected
straight-line allocation arithmetic for 64×64, 640×384, 1920×1080 and
1937×1081 and matched the formulas above. It also confirmed the framebuffer
function's sole final success read and absence of a DMA-free call.

This is static implementation evidence plus bounded arithmetic checking, not
full-function emulation, allocation-failure injection or a new hardware run.
Caller validation, sequence-result retrieval, helper semantics, recovery and
the rest of the library remain open. The
[slice-decoding conclusions](../experiments/ar9311-slice-decoding.md#interpretation)
are unchanged: these setup functions do not establish usable early regions or
resumable partial-picture output.
