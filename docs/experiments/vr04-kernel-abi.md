# VR04 kernel ABI bring-up and test boundaries

[Experiments](README.md) · [Reference board](../devices/betafpv-vr04-reference.md)

## Scope and development change

The retained MissingLynk kernel material at
`e3885c6394a7025eacd3a23f473276d9823beb25` contains both an earlier vendor-MPP
compatibility program and the later open V4L2/DRM pipeline. The compatibility
program was parked: compiling a replacement module or creating its device node
did not establish full vendor ABI conformance. This review reconciles its
bring-up notes, verification tracker and test implementations; it adds no new
hardware result.

The reference implementations include MMZ allocation, video-buffer pools,
PTS/system controls, the priority/suspend arbiter, engine IRQ forwarding and the
`/proc/umap` message shuttle. The older framebuffer replacement was removed after
its CUSE transport remained a stub; the later HUD uses a DRM overlay. The RF
driver and board services continued independently of the parked MPP program.
The radio firmware and WAVE codec firmware are separate artifacts.

The later kernel organization also separates complete added drivers from patches
to existing drivers. Clock and QSPI providers are built in; the reference camera
chain and DRM display are integrated into the kernel tree as modules. The DRM
integration needs in-tree Kconfig selections for its helpers. “In-tree” here
describes the project's build organization, not upstream Linux acceptance.

The scaler needs special treatment. A blanket reference-only list includes it,
but later notes describe standalone operation and a DMA-BUF path for recording
downscale. Its MMZ test needs `ar_osal`; the later scaler itself allocates its
internal LUT/batch storage with `dma_alloc_coherent`, and the DMA-heap test does
not need MMZ. The old dependency list is therefore not a reliable description
of every later consumer. See the [media graph](../systems/vr04-stock-media.md)
and [open pipeline](../systems/vr04-open-media.md).

## Matching a module to its kernel

The recorded `ar_osal_init` stack-protector panic occurred with the same
`6.18.36` version string but different kernel configurations. The investigation
attributed it to `CONFIG_MODVERSIONS=n` allowing the load while
`CONFIG_STACKPROTECTOR_PER_TASK=y` made the compiled `task_struct` canary offset
depend on the mismatched layout. The report describes a trimmed running kernel
with 38,882 symbols and a differently configured module build. This is the
retained diagnosis, not a newly reproduced panic or disassembly result.

The resulting build rule was to retain the exact kernel configuration, source
pin and build environment. `CONFIG_IKCONFIG` and `CONFIG_IKCONFIG_PROC` expose
the running configuration. Comparing symbol names can detect a mismatched
build, but equal names or equal vermagic alone cannot prove structure-layout
compatibility.

The reference board's `kernel0/1` image budget was approximately 6 MiB. Enabling
FB, MMC and diagnostic tracing can exceed that budget; the notes treated the
larger MMC/dynamic-debug build as RAM-only. This partition constraint belongs
to this board and must not be transferred to the Gnd485 update layout.

The MMZ test reported successful write-combined mapping of the `no-map`
carveout at `0x29200000`, allocation, a 1 MiB pattern readback and freeing. That
establishes the tested mapping path, not every cached alias or DMA-coherency case.

## What the conformance tracker actually records

The broad “Tier 0 passed” sentence is qualified by its own detailed tracker:

| Component | Recorded result | Remaining criterion in that tracker |
| --- | --- | --- |
| MMZ / `ar_osal` | Allocation, mapping, pattern readback, free; write-combined mapping succeeded | Full vendor differential coverage |
| `ar_sys` | Monotonic, microsecond-scale PTS | Timezone and GPS round trips not recorded |
| `ar_vb` | Module loaded | Pool creation, block acquisition/release and leak checks |
| `ar_sysctl` | Module loaded | Registration, query and suspend/resume behavior |
| `ar_mpp_drv` | Module loaded/probed | Registration and WAIT_EVENT woken by a real engine IRQ |
| `ar_mpp_proc_ctrl` | Module loaded | CREATE, message/write exchange and CLOSE |
| Scaler | Operation completion; later bring-up adds the pixel checks below | General filter equivalence and broader workload coverage |
| AR8030 SDIO/RF | Enumeration, firmware upload, association and reported full-rate video | Separate RF/session qualification, not proof of the MPP ABI |

Stock golden traces, complete per-module conformance, stock/open differential
comparison and the proposed end-to-end parity tier remain unchecked in this
archived program. A self-contained harness tests the recovered ABI assumptions;
a minimal vendor-library harness tests only the library paths it exercises.
Neither establishes every ioctl or the complete closed runtime's behavior.
Later [codec and DMA validation](vr04-media-validation.md) has its own evidence
and must not be erased by an earlier unchecked planning table.

## Scaler hardware and pixel checks

The scaler at `0x08840000` lies within the VO register range
`0x08810000–0x0884ffff`. The bring-up record reports that a separate exclusive
claim failed with `-EBUSY`; the implementation used a nonexclusive mapping to
coexist with VO. Its control window is the shared CGU at `0x0a100000`, including
gate offset `0x6010`, unlock/config offsets `0x6200/0x6204` and divider `0x405c`.
Mapping coexistence does not establish independent clock ownership.

The earlier verification plan calls `scalertest` completion-only. The retained
implementation and later bring-up notes supersede that description:

| Test | Actual acceptance condition |
| --- | --- |
| MMZ T1 | Every visible byte of a full-frame grayscale identity copy matches |
| MMZ T2 | Every visible byte of a cropped 1:1 copy matches |
| MMZ T3 | 2:1 downscale differs from a software 2×2 box reference by mean absolute error at most 24; output range exceeds 16 |
| DMA-BUF T1 | Exact Y-plane identity |
| DMA-BUF T2 | Exact Y/U/V identity using per-plane offsets in I420 buffers |
| DMA-BUF T3 | The same box-reference tolerance and range test, separately for each plane |
| Optional DMA-BUF T4 | Repeated roughly 2/3-size downscale timing; final Y-plane range exceeds 16 |

The bring-up narrative reports MMZ T1/T2 bit-exact and T3 within tolerance, with
the scaler completion interrupt observed. T3 is a plausibility check: the exact
polyphase filter and sampling origin lack a golden reference. Maximum error is
printed but is not an acceptance threshold. T4 checks neither each output frame
nor chroma correctness, so its throughput result is not full pixel validation.
The DMA-BUF test's existence alone is not a new hardware pass.

The source also bounds the harness claims. The DMA-BUF allocator prefers a heap
whose name contains `cma` but can fall back to another openable heap. Its
`DMA_BUF_IOCTL_SYNC` calls do not check return values. The MMZ test's fixed crop
origin `(16,8)` is incompatible with some small dimensions accepted by argument
parsing: unsigned crop arithmetic can underflow or produce an empty crop.
The default 256×128 case avoids that issue; the tool does not qualify arbitrary
input dimensions or independently prove cache synchronization succeeded.

## SD-card diagnostic limits

The block test's read mode compares two direct reads of the first 4 MiB. Its
write mode instead saves a region, overwrites it with a sector-tagged pattern,
verifies, restores and verifies again. The default region is 64 KiB near the
device end; it is not reserved scratch space and can contain filesystem data.
Invoking the tool without a mode selects both read and write.

Each I/O phase has a parent-side timeout, normally 20 seconds. A child blocked
in uninterruptible MMC I/O can survive SIGKILL. A timeout therefore bounds the
parent's wait, not the kernel request or the time at which storage stops changing.
The initial device-size query is also outside those phase timeouts.

The default backup is `/run/sd_rwtest.bak`, ordinarily volatile. The backup writer
closes the file but does not fsync it. Consequently, the suggested power-cycle
then restore procedure does not by itself preserve the original bytes. The
backup format stores a magic, offset, length and data, but no card identity or
checksum; restore does not compare the saved region with the selected device's
size. These are limits of the diagnostic procedure, not new card-failure results.

The [board study](../devices/betafpv-vr04-reference.md) retains the actual SD
clock/phase and GPIO findings. Historical RAM-only loading instructions and
claims of automatic reboot recovery do not override the recorded kernel panic,
SDIO warm-reload hang or storage-restoration limits.
