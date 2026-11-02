# AR8030 first uplink, AP login and bounded TX recovery

[Experiments](README.md) · [First received packets](ar8030-receive-acquisition.md)
· [Later receiver integration](open8030-early-dev.md)

These Gnd485 18.21.10 DEV trials were recorded on 14 October 2026 UTC. Trials
97–107 connect checked MAC reception to native transmit storage, AP assignment,
compact control and internal command streams. They precede native host-socket
delivery and video. Later [Open8030 validation](open8030-validation.md) establishes
additional behavior; the limits below describe these historical experiments.

Most monitors lasted ten seconds on channel index 24, 5,770,000 kHz. Acquisition
usually took about seven seconds, leaving only about three seconds of observed
traffic. The supplied 42-entry channel table was configuration context, not
evidence that these fixed-channel experiments swept or hopped through it.

## First admission and three local transmit completions

Trial 97, operation 54, received one valid 48-byte control packet at
7,076,887 µs, admitted the configured/candidate peer and completed the TX-engine
handshake at 7,077,603 µs. Three pre-encode requests produced three submissions
and three TX-end events, with no errors or retained packets. The last submission
was at 7,101,330 µs, long before the 10,000,020-µs monitor ended.

The final transmitted packet was an eight-byte MAC header plus a copied 16-byte
DEV report. It reported state 1, **assigned slot 8**, receive-channel index 24
and no extensions. Header CRC16 was `3ba5`; report CRC16 was `ca90`. Those
checks reproduce from the retained bytes and original table. Selecting physical
TX context 0 did not assign over-air slot 0. TX-end validated local ownership
retirement; neither AP acceptance nor a native linked session was established.

The transition initialized the mode-zero seven-byte PHY control block, selected
the physical buffer, cancelled search, set peer identity and performed a bounded
TX handshake. Different RX/TX channel indices were rejected; a failed handshake
left TX held. Initial raw status inputs were explicit zero hypotheses rather
than sampled dynamic gain state.

An offline follow-up also rejected equating link state 1 with PHY policy 1.
The original command handler at `0x23759a` decodes a requested policy before
calling `0x2371e4`. Its upper-byte change to RX8 is not an unconditional
admission-side write. Truncated pseudocode around a misclassified division
helper had obscured that gate.

## Correcting the acquisition TX profile

Trial 98, operation 55, added the source-18 write to `B4C` bits 3–7 using
`(raw_user << 1) | 1`, with the user from `F68` bits 4–7. All 413 events selected
user 0, leaving `B4C = 8`. Four received controls and three TX completions still
did not explain or fix the stall; the tested register value was unchanged.

The working-stock acquisition reference exposed a separate mismatch:

| Field | Earlier native profile | Corrected startup / stock reference |
| --- | --- | --- |
| `B00` after release | `14086021` | `14093021` |
| `A00` low half, TX0 duration count | `168f` / 5,775 | `0da3` / 3,491 |
| Derived six-command frame | 7,096 µs | 5,634 µs |
| Search dwell at alternating wraps | 9/16 ms | 7/12 ms |

Addresses are offsets from `0x0a110000`. The original MCS table at `0x25412c`
maps internal index 1 to key 1. With the recorded bandwidth/interleaver/mode,
the TX calculation selects modulation 0, code field 3 and rate field 1.
The old fields corresponded to internal index 2. These are internal TX indices,
not an inference from GET_STATUS's receive-MCS field.

The original TX duration calculation was:

```text
floor(((17/16 × (12 + 2)) + 3) × 2048 / 22.4) + 600 = 2234 µs
round(2234 / 0.640) = 3491 counts
```

The added 600 µs combines an ordinary 200 with user 0's additional 400. The
other slot's high half of `A00` is preserved. One historical startup note gives
the old dwell as 9/14 ms; the recovered dwell formula and 7,096-µs frame instead
give 9/16 ms. The corrected trial's final 12-ms readback agrees with 7/12 ms.

Trial 99, operation 56, changed the MCS fields and duration together. It received
403 valid MAC/full-control packets and completed 200 TX packets, with last RX
at 9,995,785 µs and last TX at 9,997,895 µs. This sustained control traffic for
about 2.93 seconds after acquisition. It did not isolate the effects of the two
changes or implement adaptive MCS.

The last retained control still advertised a free slot without login records.
That single retained message could not establish that all 403 lacked login.
Trial 100, operation 57, closed that observation gap: all 395 evaluated controls
returned WAIT_LOGIN, all advertised slot 0 free, and none contained extension 0.
It completed 196 reports but still transmitted state 1/slot 8. First-login and
first-matching buffers remained empty. Local completions alone did not establish
that the AP had accepted the uplink.

## A stock report reference, with a corrected status claim

Trial 101 ran **instrumented stock**, not the replacement firmware. One
instruction at `0x246146` redirected the bounded GET_REG reader to RAM page
`0x282000`; reads sampled the 16-byte report at `0x282e00`. The volatile stored
candidate-list command was supplied after two seconds. The reader retained its
256-byte request and 4 KiB page bounds. RAM samples were not synchronized with
the report writer.

Among 970 samples, 648 preceded report construction, three contained valid
state-1/slot-8 reports and 319 contained valid state-2/slot-0 reports. Every
constructed report passes the original-table CRC16. The first states appeared
at 8,303,266 and 8,319,633 µs. Their 16,367-µs separation is a sampled RAM-state
interval, not RF or end-to-end latency.

| Report field | Stock state 1 | Stock state 2 | Native trial 100 |
| --- | ---: | ---: | ---: |
| Assigned slot | 8 | 0 | 8 |
| Receive-channel index | 25 | 25 | 24 |
| Six-bit slot value | 0 | 25 | 0 |
| Report byte 6, high two bits | 0 | 0 | 3 |
| Report CRC16 | `84f7` | `5e50` | `ca90` |

The historical native encoder reproduced both stock forms byte-for-byte when
given their recorded inputs. During this documentation review, independent
encoding reproduced the constructed samples again and the observer binary was
confirmed to differ from stock only in that instruction. Raw inputs 2/2 give
the stock's two zero status bits, but many raw values do so. These bits cannot
identify the entire raw status bytes or establish dynamic LNA control. Channel
25 belongs to that stock run and must not be copied into a channel-24 plan.

The probe emitted GET_STATUS bytes 0–35, while the link entries start at offset
252. An early derived status interpretation was removed. Those snapshots provide
no slot-state evidence; the state comparison above rests on CRC-checked report
RAM. This report reference made another wire-format guess unnecessary, while
leaving the physical uplink and gain setup to investigate.

## Gain correction produced matching AP login records

Trial 102, operation 58, changed four effective TX0 gain fields to match the
configured startup index 25 and working stock: path A analog/digital 23/78,
path B 22/78. The prior hardcoded fields matched the calibration row labelled
`14dbm`. These labels and register gains are not independently measured output
power. External calibration, factory/band offsets and clock scaling were
resolved before constructing the diagnostic plan; no calibration table needs
to be embedded in the public documentation.

Original setters program `9A4` bits 8–13 and 0–7, `DA4` bits 0–5 and `DA0`
bits 24–31. Readbacks were `9A4 = 88cc174e`, `DA0 = 4e0000cb`, `DA4 = 16`.
Report status inputs remained raw 0/0, preserving that preceding hypothesis
while changing the gains.

Of 597 checked controls, seven waited for login and 590 contained matching
local-identity/token records assigning slot 0 with AP state 2. The first match
arrived at 7,081,313 µs. Native code still only observed assignment and reported
state 1/slot 8. It completed 27 transmissions, then stopped at 7,212,360 µs;
RX continued until 9,627,214 µs. This supports the startup-gain correction in
the tested setup without establishing a general RF-power calibration.

The last DMA was 80 bytes but only 64 were retained. Its complete 12-byte
header passed CRC and declared a 20-byte port-0 payload. Only four payload
bytes remained in the prefix. A checked header or separately retained full
control is not a complete captured command payload.

## Applying assignment, synchronization ACK and compact control

Trial 103, operation 59, applied a matching mode-zero/slot-zero login only with
AP state 2 and supported zero remote-sync flags. It queued command 4 containing
the lookup check of all four synchronization bytes. Nonzero flags were not
acknowledged as executed. The report's six-bit value became configured power
index 25, distinct from the AP token and analog gain.

The copied ACK packet was 33 bytes: 12-byte header/descriptor, 16-byte report,
five-byte command envelope. Because the raw budget is rounded down to multiples
of eight, it requires at least 40 bytes. An insufficient budget preserves the
pending ACK while allowing a report-only packet. Header, report and payload
have separate DMA segments. The encoder writes the unused descriptor nibble
as zero; stock residue in that nibble does not establish fragmentation flags.

The local transition sets `B4C` bits 3–7 to 1 and `CF4` bit 21. A stale token
or profile stops TX; automatic reacquisition and full PHY policy were later work.
Login evaluation matches the already accepted peer, preserves bitmap slot
indices across holes, and requires the record token to match the slot word's
high byte. Stock's twelve-byte scratch stride is not padding in the ten-byte
wire login record. Assignment by itself does not establish state 2: remote
synchronization and AP-state checks remain separate gates.

| Measurement | Trial 103 | Trial 104, compact decoder added |
| --- | ---: | ---: |
| Valid MAC packets | 534 | 250 |
| Valid full controls | 12 | 13 |
| Compact controls | 522, unsupported by runtime | 237, validated |
| Native state / assigned slot | 2 / 0 | 2 / 0 |
| Sync ACK submissions | 1 | 1 |
| TX submissions / completions | 26 / 26 | 26 / 26 |
| Last TX from monitor start | 7,222,723 µs | 7,209,733 µs |
| Frames with payload descriptors | 484 | 241 |

The retained trial-103 compact control passed an independent offline check even
though that runtime did not yet decode it. Trial 104 validated compact messages
against the existing slot/token context; their lack of identity, mode or channel
fields makes them unsuitable for idle admission. Compact parsing did not remove
the roughly 150-ms post-acquisition TX stall. Different RX totals alone do not
isolate a cause.

Trial 104 observed port-0/3/4 payload totals of 4,180/6,547/16,504 bytes. Its
complete first port-3 payload contained 76 bytes, including OSD text
`BATT < FULL`. This is content evidence, not independent payload integrity or
video delivery. MAC ports are not host SDK socket numbers. The final DMA was
136 bytes with only 64 retained; its complete checked header declared port 0
length 20 and port 4 length 92, not a fully retained 92-byte port-4 payload.

## Request-ID recovery and stream reassembly

Operation 61 ported the DEV request-ID recovery branch at `0x231372`. Source 18
supplies four-bit IDs from `F64`; the second exists only when `B00` bit 4 is set.
With configured threshold 6, repeated failure to advance by the expected one
or two steps triggers a disable/re-enable cycle across subsequent events.
Both comparisons use the prior ID, even if the first comparison advances it.
The field setter changes only `B00` bit 30, with inverted enable polarity.
It does **not** retire DMA-owned buffers or implement a general software ARQ.

| Observation | Trial 104, before | Trial 105 | Trial 106 |
| --- | ---: | ---: | ---: |
| Valid MAC/control packets | 250 | 592 | 329 |
| Compact controls | 237 | 580 | 317 |
| TX submissions / completions | 26 / 26 | 134 / 134 | 131 / 131 |
| Last TX from monitor start, µs | 7,209,733 | 9,998,537 | 9,987,587 |
| Recovery disable / enable counts | — | 55 / 55 | 57 / 57 |

Both new trials reached state 2/slot 0, submitted one ACK and recorded no TX
errors or retained packets. Frequent recovery cycles still indicated incomplete
link/PHY behavior. The bounded result is continued local transmission through
the monitor window, not endurance or proof of delivery of every packet.

Operation 62 added a 132-byte internal command-stream buffer. Command envelopes
can span MAC descriptors or share one descriptor. A missing magic causes
resynchronization; a failed XOR check discards the consumed envelope. Trial 106
decoded 302 command-2 envelopes where the frame-local diagnostic recognized only
171, with zero checksum/length errors. Its accounting closes exactly:

```text
6072 received = 302 × 20 command bytes + 24 discarded + 8 buffered
```

The 24 discarded bytes are observed resynchronization loss, not independently
proven RF corruption. Trial 107 repeated this path while adding a socket-related
instruction: 409 valid packets, 136 TX completions, 56 recovery cycles and
`5868 = 292 × 20 + 12 discarded + 16 buffered`. Command-2 measurement policy
and host-socket delivery remained separate implementation tasks.

The diagnostic plan gained effective gains (`DEV1`), report power index (`DEV2`)
and recovery threshold (`DEV3`). The latter accepts 0–127, with zero disabling
recovery; stock separately maps negative signed configuration values to zero.
By trial 106, RX-info occupied 1,021 words / 4,084 bytes, leaving only 12 bytes
under its 4,096-byte response cap. This motivated later bounded retrieval rather
than indefinitely appending diagnostics.

## Verification and remaining boundaries

Original-runtime addresses above are bound to SHA-256
`3e454131a334ec422c0b2e8ae4924d6bda68cb61b70fd754da26412578b39c87`,
load base `0x204000`. Synthetic ELF mappings and partial instruction models are
analysis metadata. Historical sanitizer suites covered ring/sequence ownership,
login bitmap/token cases, compact controls, request wrap/order and command splits.
Those suites and physical trials were not rerun during documentation review.

Read-only checks here reproduced 12 retained TX/report or ACK packet checks,
full/compact controls from trials 103–106, both truncated DMA-header extents,
all 322 constructed stock-report CRCs, and the single-instruction stock observer
patch. The four retained ACKs were byte-identical copies; they are separate
trial artifacts, not four distinct wire-format examples. The trial-103 ACK is
SHA-256 `11111cedf06f7ea137b3bb46220dd50b90d5bb568835e7fc7717e43c3df4194f`.
Packet checks do not validate unretained intervening payloads or establish RF
receipt from a local TX capture.

Recorded stock restoration used a full receiver reboot, 24 getters and a viewed
direct capture. Trial 97's first capture included 47 boot-transition black
frames; the settled capture had 301 nonblack frames. Most later measured
captures also had 301 nonblack frames; trial 104 had 302. Warmup errors remained
separate from the measured interval. These establish stock recovery, not native
video. Full PHY maintenance, payload sockets and Linux/backend integration were
still required after this milestone.
