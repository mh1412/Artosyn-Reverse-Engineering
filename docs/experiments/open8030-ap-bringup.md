# Open8030 AP bring-up: held preparation, transport and early soak

[Experiments](README.md) · [Release validation](open8030-validation.md)

The hidden October 17, 2026 records distinguish several milestones that a
single “AP works” statement would obscure: offline role defaults, native SDIO
execution, held radio preparation, host-write ownership, generated-data transfer
and encoded-video progress. Later release validation supersedes the early lack
of AP integration, but does not erase failed candidates or convert a short
throughput test into sustained video qualification.

## Matched stock input

The Sky482 18.21.10 radio container `bb_demo_sky_3v3.img` was 428,544 bytes,
SHA256 `91f4dbd0a13bd4c18dac7dd832fdbbbe929f90a2eb5a186b68252c834efa6105`.
The recorded live configuration hash was
`bf13ab10b0c0a0facce2e84374d96dfa341cdd549f114e1279992c6626f7a043`, and
SDIO module hash `125b063807fa943398e5f13bd6efde90436c3a209791847434264edaed1df111`.

| Container region | File offset | Bytes | Load address | SHA256 |
| --- | --- | ---: | --- | --- |
| Bootstrap / `troot` | `0x200` | 944 | `0x0a098a00` | `a9e59b0e05c42042e27c35ca998c83326aa9d8ab0cea294bc2d0f7199929df68` |
| Configuration / tool field `signature` | `0x600` | 16,384 | `0x00200000` | `010fa4da113619a5330d247e2cfffe73381ade39ae5b1a707c946db4bca2d18f` |
| Runtime / tool field `spl` | `0x4600` | 410,368 | `0x00204000` | `bb209e4a6f10b7928f67baf6aaf4da2e1e3b21e5832ca782481c27c7e5833cea` |

The payload ended at byte 428,288 with 256 trailing alignment bytes. The
analysis explicitly recorded container checksums as unverified. Its synthetic
RISC-V ELF used an analysis-selected ABI/permissions/entry, not recovered source
ELF declarations. Region mappings alone do not prove boot order or simultaneous
residency, and the `signature` field name does not establish cryptographic signing.

## Offline AP versus DEV defaults

The defaults comparison used the same Sky image with raw mode 0, clock flag 1
and chip profile 1. Of 704 writes, 23 differed by AP/DEV role; Sky DEV output
matched the original receiver DEV output under those inputs. This was an
offline helper comparison, excluding GPIO, calibration and RF execution.

The differing register offsets and values were:

| Offset | AP | DEV |
| --- | --- | --- |
| `800` | `00000297` | `00000217` |
| `808` | `020003ff` | `030003ff` |
| `80c` | `00980001` | `00000001` |
| `810` | `133a2109` | `2309113a` |
| `8a0` | `00980000` | `00000090` |
| `900` | `0000dc92` | `00000000` |
| `904` | `00000000` | `00dc9200` |
| `910` | `00000000` | `0000dc92` |
| `938` | `dc920000` | `00000000` |
| `970` | `0000168f` | `168f0c35` |
| `98c` | `000f0000` | `00010000` |
| `9a0` | `3c30a344` | `3c300345` |
| `9a4` | `02000000` | `40000000` |
| `9e0` | `3c300000` | `3c30a344` |
| `9e4` | `40000000` | `02000000` |
| `b20` | `140a4173` | `14080173` |
| `b54` | `40000000` | `00000000` |
| `b70` | `00200000` | `00000000` |
| `cec` | `42000000` | `40000000` |
| `cf0` | `00030002` | `00030000` |
| `d20` | `00000000` | `b03d0b34` |
| `d24` | `00000000` | `fff83902` |
| `d2c` | `00000000` | `1a000000` |

All values are hexadecimal. These are defaults-stage values, not a complete
live-state register image or instructions to overwrite a running radio. The
packed AP defaults hash was
`b3bcf3b2bbc88af903ef6129b58827af4d8432c0f777657c5977551d7956b931`.
The initial validation recorded 67 native suites and 18 Python tests passing,
but `ap_runtime_integrated=false` and `firmware_deployed=false` at that point.
Its all-black capture, including UI, did not verify a visible camera scene.

## Native transport and sector padding

Commit `0d921fa` produced a 76,552-byte runtime with hash
`62fbfb76eba1430bc0792dd74c63e22376a93d657bbdc91f27185ddccd35fb74`
and a 94,720-byte container with hash
`3bb998645ff387067728377858a958ced2c01a29bcd37b259f4a23c0df0f4e53`.
The stock AP configuration was deliberately rejected with `-95` before RF
initialization. The native identity explicitly reported RF unsupported.
Successful identity/probe transactions proved transport execution, not AP video.

A direct defaults request exposed a host-write boundary: the unpadded
1,043-byte transfer returned only 1,024 bytes. Padding to 512-byte sectors and
handling short writes allowed all 704 defaults, three calibration rounds
(status 6), and PSRAM result `0xaa991007` / error 0 to complete. The earlier TCP
request timeout alone had been inconclusive. These stages did not start RF.

## Held AP preparation and large host writes

Commit `a6d50a3bc8f50ab204634ef7f0c94bdd658ed398` produced:

| Artifact | Bytes | SHA256 |
| --- | ---: | --- |
| Runtime | 82,128 | `a8f63bd31abc48da38d39ec85f2a99723353c0ec3633eefe2e39442dd46c6ee8` |
| Container | 100,352 | `8c2ae0721906cfa251d9f9533889f08deacc131d04425306947afe7b38574c17` |

The held trial prepared 5,770,000 kHz with gain/amplifier values
`37,76,37,76` derived from live stock TX register readback. Identity came from
the matching configuration fallback; it was not asserted to be OTP identity.
The diagnostic reached phase 6/error 0 and register `800 = 00080297` without
requesting RF start.

It admitted a complete 10,240-byte SDIO host-write frame containing 10,213
application bytes. The client required asynchronous completion before passing;
admission and completion here concern host-buffer ownership, not RF delivery.
The implementation checkpoint had five retained message slots, two pending host
writes and immediate parser-byte copies. RF preparation copied into immutable
TX storage before releasing retained host slots. Queue-full rejection `-263`
did not advance the application position. These bounded semantics were not full
stock RTOS/backpressure equivalence.

The first diagnostic IDs collided with existing commands: old AP_INFO
`0x0c5a002b` returned 596 bytes of telemetry, not AP status. The corrected
development IDs moved prepare/start/info to `0x0c5a0030/31/32` in `d929648`.
A stale checkpoint paragraph still retained the old IDs; the explicit collision
record and later correction take precedence. The held trial did not validate
the corrected status/start path or normal AP host lifecycle.

An independent timed reboot restored stock. The recorded boot changed, temporary
trial files disappeared, stock image/config/module hashes matched, and both
radio states returned to 2. The accompanying report describes fresh HDMI camera
and stock overlays. That image has not been reinterpreted as a latency or
controlled scene-freshness measurement here.

## Early ownership faults and the 1-KiB-copy checkpoint

Subsequent trials exercised broadcast lengths, coding state and RX ownership.
The archived checkpoint records include:

| Candidate | Observation | Limit of the conclusion |
| --- | --- | --- |
| Deferred RX, long-48 | Stalled about 266 seconds into a 300-second soak | Initial video did not establish sustained progress |
| Pending-DMA diagnostic, long-48 | 18,208-byte DMA remained pending with no video progress | Pending metadata helps locate the stall; it is not a completed transfer |
| Short-24 variant | Video stopped at 47.675 seconds; pending length 9,104 | Shorter budget did not remove the failure |
| Eight-byte headroom experiment | Stalled on 2,346 bytes, radio daemon disappeared while SSH remained | Change was reverted; broad reliability not established |
| Corrected metadata order, matched MCS 8 | Short transfer received 23.402 of 30.466 offered Mbit/s, with invalid packets/drops; later host CMD53 timeouts | Correcting field roles did not close the transport failure |

These failure summaries are preserved from the handoff record; their entire
raw trace populations have not been independently re-decoded for this page.
The later guard deferred coding resets while RX DMA owned its buffer and
coalesced requests until real completion. Tests first reproduced the old
ownership failure, then passed with the guard.

Commit `3bb0494271b2a0d3088c40e4ed4400ebc9c793f0` used four host output blocks,
1,024-byte host-copy chunks, deferred RX/coding reset and copy-carry handling.
Its container hash was
`d1b613846d61ad9bf1276e1d2add782eb31cba2aec9777a79af55683e0f803b1`;
runtime hash `6f82ebb1ddb211ed88d699793cd55dd66729b3f7c2d15b5e9bd5eb0ba50d9a95`.
The configuration used 40 MHz, broadcast length 48 and AP MCS 8, with automatic
DEV MCS initially 1. The manifest explicitly calls this a mismatched-coding
ownership test, not a recommended fixed-AP configuration.

Two different measurements must remain separate:

| Measurement | Recorded result |
| --- | --- |
| Short generated-data transfer | 862 blocks of 49,152 bytes; 42,369,024 sent/received exactly over a 10.009054-second sender interval, 33.864558 Mbit/s |
| Encoded-video progress sample | 294 samples over 299.284678 seconds; all connected, 459,005,535 video bytes, 12.269403 Mbit/s, every sampled interval advanced |
| Video-sample counters | Zero endpoint drops, AP command errors, skips and TX errors |

The generated-data benchmark paused the camera. Its `exact=true` result and
rate come from the saved summary; they are not an independent byte-by-byte
replay here, a sustained 35-Mbit/s result, or 35-Mbit/s encoded video. Likewise,
the five-minute zero-drop sample does not establish universal losslessness.

## 2-KiB candidate and USB disappearance

Commit `8e90a33054c502c57b568b114e16cee2546d8a61` changed the copy chunk to
2,048 bytes. The audit found identical probe, AP defaults and coding-plan
inputs; startup-template differences were the expected image hash. USB
disappeared about 47 seconds after launch, before throughput testing and before
the fallback deadline. The VRX retained its image; VTX runtime became unknown
after disappearance and a PSU cycle. `loaded=true` in the manifest does not
override its later `retained=false` and `throughput_test_run=false` fields.

The host port remained authorized/enabled. Causation was explicitly unproven:
temporal proximity does not establish that copy size caused USB failure.
Connected USB VBUS also meant PSU-off alone did not establish full power removal.
Later normal-mode recovery restored root access and stock camera/radio processes;
the saved helper-script hashes matched the earlier repaired stock files.

The [October 22 release bench](open8030-validation.md) is a separate accepted
pair with its own hashes, drops and recovery observations. Its success supersedes
the early “no AP runtime” stage without converting these failed candidates or
the untested 2-KiB throughput configuration into accepted results.
