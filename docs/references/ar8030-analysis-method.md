# AR8030 analysis inputs, synthetic ELF and evidence limits

[References](README.md) · [AR8030 study](../soc/ar8030.md) ·
[Validation history](../experiments/open8030-validation.md)

The initial reproducible workflow was recorded on October 12, 2026, for matched
17.5.8 GND and SKY images. Later receiver reconstruction used Gnd485 18.21.10.
These versions have separate address maps and image identities. The workflow
extracts and analyzes supplied firmware; it does not itself execute that firmware
on a device or establish replacement compatibility.

## Identify the input before interpreting an address

| Input | Complete-image SHA256 |
| --- | --- |
| GND 17.5.8, 424,960 bytes | `dbdea6c72bb0d5f75ffa395693720967b19c8993f0ad6419083f49a3c1ac3705` |
| SKY 17.5.8, 423,424 bytes | `e66dae0ab8d9ecb248239af45f298dc334ad602aa6f5f4700a92b299a51bb51b` |
| Gnd485 18.21.10, 431,104 bytes | `dc58247910af564376350404383c5c411cbec21663873fba24e70c61a3584e3c` |

The 17.5.8 extraction manifests retain these complete-image identities. Their
main payloads differ: GND is 406,720 bytes, SHA256
`9f4b661bf8b8cecc4dd989d0c9b7d22f32253512784e06f2f3f51816806c5b49`;
SKY is 404,992 bytes, SHA256
`bd452d76f15c9fb12ef506acd3912b83970068ba67695bc3aa135c4d7ff76135`.
Both mains load at `0x204000`, which does not make internal addresses
interchangeable. The corresponding Gnd485 18.21.10 main is 412,736 bytes, SHA256
`3e454131a334ec422c0b2e8ae4924d6bda68cb61b70fd754da26412578b39c87`.

Executable payload, embedded configuration, external loader configuration and
board calibration are separate inputs. In particular, embedded template defaults
are not proof of the DEV settings supplied by the Linux loader. Preserve their
individual identities and the role/board that consumed them.

## What the extraction checks establish

The reviewed parser accepts magic `0x41528030`, image type 1 and a header at
least 52 bytes long within the supplied file. It rejects an empty main, truncated
segments, 32-bit destination overflow and overlapping nonempty download ranges.
It rounds transfer boundaries to 512 bytes and accounts independently for
present bootstrap and configuration segments. Missing bootstrap handling follows
the observed Linux transfer layout; it does not prove ROM acceptance of such
an image. See [loading and driver lifecycle](../firmware/ar8030-loading-and-driver.md).

The last payload's transfer padding may be absent. Conversely, extra bytes are
reported rather than assigned an invented meaning. GND 17.5.8 has 320 bytes
after its main payload, ending on the aligned boundary; SKY has an additional
512 bytes after its aligned payload end. This parser does not authenticate the
header checksum or opaque trailer. A caller-supplied whole-file hash checks
identity, not signature validity or device acceptance.

Embedded configuration beginning with `{` is decoded before writing outputs.
The parser uses a JSON decoder that can stop after the first complete object,
so this is not a claim that every remaining configuration byte is JSON. It
refuses an existing output directory; malformed input or hash mismatch does not
replace previous extraction evidence.

## Synthetic ELF is an analysis convention

The original download images contain no vendor ELF declaration. The workflow
wraps unchanged bootstrap, configuration and main bytes in an ELF32 little-endian
RISC-V container at their header-provided addresses. It adds `_start` and optional
`_bootstrap` symbols, section names, permissions and nonloaded provenance data.
Main code and data are mixed; the generated permissions are not recovered memory
protection settings.

The recorded REA raw-binary interface did not admit RISC-V. Its new ELF path
required a double-float declaration and explicit generic RV32 selection; ordinary
automatic import could choose an Andes variant. The wrapper therefore uses
ELF flags `0x5` and analysis language `RISCV:LE:32:default` with compiler
convention `gcc`. **This does not establish a double-float firmware ABI, compiler
identity or the complete ISA extension set.** The derived ELF is not a loadable
radio-firmware package.

The wrapper infers uninitialized storage only after recognizing the main
startup's AUIPC/ADDI register pairs and exact zeroing-loop pattern. GP must equal
the end of the main payload, and the inferred end must be within the checked
extension bound. Both matched 17.5.8 mains yield a 23,040-byte extension. This
comes from startup instructions, not a recovered linker map; an unfamiliar
startup is rejected rather than assigned the same layout.

The composite mapping also combines boot phases. Bootstrap scratch can alias
addresses later used by configuration. Showing both in one analysis database
does not establish simultaneous residency, execution order or destruction of
live configuration. The [boot reconstruction](../firmware/ar8030-boot-and-lifecycle.md)
and [source-bootstrap trials](../experiments/open8030-early-dev.md) provide
separate evidence for actual startup behavior.

## Pin the tool implementation and verify its observations

The recorded Linux x86_64 setup used Python 3.12, Node 22.22.0, npm 10.9.4,
Ghidra 12.1.4 and Temurin 21.0.12.1+1. These are historical tool identities,
not recommendations about current releases.

REA was built from commit `653067d799591ce2b8fdc84e99c160ac17e2860f`.
That source build labels itself 6.4.0, but the npm 6.4.0 release did not contain
the RISC-V support used in this study. The version label alone is therefore
insufficient. The session driver checks the source commit; tool preparation
checks pinned archive hashes. Its configured 1 GiB Java heap is not a limit on
the complete process family's memory use.

A persistent REA MCP session per image discovers the available schemas, opens
the Ghidra provider, inspects actual language/compiler/source mappings, reads
entry bytes and records entry/callee/bootstrap results. Additional query plans
are restricted to named read operations. The original 17.5.8 plans must not be
reused unchanged against another image merely because its entry address agrees.

Retained evidence includes the selected schemas, requested and observed profile,
source digest, query results, completion receipt and close receipt. Closing a
successful session saves a query-cache snapshot and removes its temporary
Ghidra project. That snapshot is not a persistent Ghidra database. A failed
import, query or cleanup remains a failed run with its diagnostics.

The summary checker verifies agreement among the extracted ELF hash, manifest
and completed-run receipt; observed segment addresses, lengths and hashes;
unchanged original/modified source identities; and the retained 64-byte entry
read against the main payload. These are explicit observed checks, not a full
independent attestation of the provider's ELF loader.

Its procedure and string counts include only addresses inside the main payload.
The retained board-init runs contain 1,048 procedures / 2,009 strings for GND
and 1,041 / 1,995 for SKY. These are recovered inventories, not source-level
function counts, resolved-function counts or a percentage of semantic similarity.

## Generic decompilation and the partial instruction model

The later 18.21.10 work requested the partial custom language
`RISCV:LE:32:AR8030_XThead_partial` through REA and received
`unsupported_target`. The provider was not patched to bypass that restriction.
Generic REA supplied call/reference evidence, while separate Ghidra runs using
the partial model corroborated custom-instruction behavior.

Generic analysis misinterpreted custom operations and marked division helper
`0x24daea` non-returning in relevant paths. Consequences included truncated
callbacks, disjoint function ranges and missing paths beyond logging calls.
Those pseudocode omissions are not evidence that the original path cannot run.
GP seeds, function boundaries and returning-function annotations are explicit
analysis inputs, not vendor declarations. A restore tail such as `0x24dbea`
cannot simply be labeled a returning function if it is not a separate function
in that database.

Custom-model use checks both source and installed model hashes, including the
compiled model. A model update invalidates assumptions tied to older decoding;
the generic and partial-model results need their own identities. Successful
export without a `custom0` placeholder is not enough to mark a function resolved.
For example, one multiply-accumulate lead was later identified as `th.mula`,
while physical tests of two byte-swap words establish those exact operations,
not a complete CPU/ISA identification.

GNU disassembly checks also need the correct section: an embedded binary may
require `objdump -D` rather than ordinary symbol-driven `-d`; the analysis ELF's
main is in `.spl`. Compare the instruction bytes at the claimed load address,
not just the printed mnemonic or decompiler's inferred argument types.

## Receipt validity is separate from result completeness

The evidence wrapper checks input digest, provider/language, evidence ID and
presence of a result. Composed traces may carry an upstream profile and place
their result in `normalized_result` while `raw_result` is null. That is a valid
representation, but a valid identity-bound receipt can still contain a truncated
graph or unsupported operation. The validator deliberately accepts a synthetic
truncated-result fixture; it checks identity, not complete semantic coverage.

For an original-instruction comparison, record the executed ranges, concrete
initial state, native result and every intercepted helper. A synthetic MMIO bank,
stubbed allocator or replaced queue callback bounds what a passing result means.
Preserve failed controls and compare intermediate fields/effect order where a
matching final total could hide cancelling errors. Native unit tests, original
instruction comparisons, runtime integration and physical acceptance remain
different evidence levels.

Historical handoffs need the same care. The review reconciled 46 distinct
versions by reading the complete retained history and every added/replaced
passage, checking hashes to account for unchanged copies. Repeated “current”
headings, unused trial numbers and memory totals belonged to their individual
checkpoints. In particular, the final offline build's unused-trial statement
was superseded by [physical trial 162's failure and trial 163's baseline
revalidation](../experiments/open8030-baseline-revalidation.md). A later successful
run of an older runtime must not be attributed to the newer implementation.

During this wiki review, eight extractor tests and two evidence-binding tests
passed. Rechecking both retained 17.5.8 board-init runs confirmed mapped payload
hashes/addresses, the 64-byte entry reads, ELF program-header bytes and the
23,040-byte startup-storage inference. This reused saved provider observations;
REA/Ghidra, firmware emulation and hardware were not rerun.
