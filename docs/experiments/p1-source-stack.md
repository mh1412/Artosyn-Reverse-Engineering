# P1 source-stack validation

[Experiments](README.md) · [P1 Air](../devices/betafpv-p1-air.md)

Research dates: 2026-10-31 through 2026-11-02 UTC. These stages explain which
earlier limitations were resolved and which remained after direct installation.

## Milestones and evidence

| Stage | Result | Boundary |
| --- | --- | --- |
| Stock inventory | AR9311, 256 MiB layout, SC231AI and separate radio verified | Official generic DT size was not treated as fitted RAM |
| First RAM boot | Source Linux 6.18.36 and USB management booted; cold cycle restored stock | Exact-build handoff, initially CPU0-only |
| Source camera | 6,000 frames / 100.051 s = 59.969 fps; all decoded | Extracted tuning/microcode still required |
| Camera module reload | 600 frames at 59.598 fps, visible content after reload | Fixed VIF AXI restart path; unrelated historical slowdown unresolved |
| Lit-scene AWB | 1,200 frames at 59.531 fps; neutral-looking regions improved | Not a calibrated color-chart or all-illuminant result |
| Matched radio integration | 3,473 consecutive frames, 22,206,988 HEVC bytes, no framing errors | Matched Open8030 revision and board configuration required |
| Integration restart | 1,079 frames with complete host decode | No claim of uninterrupted playback through every failure |
| Physical controls | Bind release, red/green states and long-hold behavior observed | Link/LED tests did not repeat full-stream decode |
| Persistent settings | Saved peer/preference survived stock and source boots | No exhaustive mid-write NAND power-cut testing |
| Supervision | Application/helper crash cleanup and bounded restart worked | Receiver setup repeated after crash |
| Direct NAND boot | Initial install, native update, cold boots and stock return passed | Preserved stock bank; not A/B source-firmware rollback |

## Radio mismatch and framing

An early Open8030 pair associated but lost or duplicated records under bulk
downlink. Updating to matched revision `3829041`, including the AP/DEV width
switching change, produced three byte-exact downlink transfers: 1,608,000,
12,288,000 and 24,576,000 bytes. The last followed an air-unit power cycle and
receiver radio reset.

A reverse-direction startup test still missed record 129 of 500, while a fixed
5 MHz reverse test delivered all 500 exactly. Downlink success therefore did
not establish all-mode bidirectional integrity.

Kestrel integration also initially suffered repeated link drops with an older
receiver build. Drops disappeared after loading the matching validated firmware.
This is an observed resolution, not a complete regression bisect.

The stock P1 SDIO driver uses an optional DD block-tail envelope. Native arlink
uses its own framing; host transport and firmware mode must match. In addition
to boot configuration, Kestrel requires runtime peer selection. An embedded
candidate in the radio JSON alone did not establish the application session.

## Decoder agreement

Receiver Wave521C hardware decoded an eight-frame camera sample to the same
I420 bytes as the host decoder:

```text
3a17c5cefb31fac1b39ba148b00c59e745fae7e8c67f725627c13bd6be89fc90
```

A later 16-frame integrated-stream sample also matched:

```text
6e99f27b900d4f7a55431e16eb181b2a5597a647ab72d3fd435536d8e3ce13b7
```

HDMI was disconnected for the earlier decode test. Pixel agreement does not
establish display operation or end-to-end latency.

## Failure interpretation

- Valid frame counters accompanied black output before the VIF restart fix;
  pictures must be checked alongside counters.
- One run filled the RAM filesystem before its intended frame count. It was
  not accepted as a completed long recording.
- Early DMA-BUF output corruption disappeared after codec reset, but the exact
  initial cause was not established. Later clean/cold tests provide the accepted
  evidence rather than a speculative explanation.
- A receiver harness requested eight frames while supplying sixteen, and another
  attempt filled temporary storage. Corrected retries, not those failures, support
  the decoder-agreement result.
- Timed captures can end inside a packet. Complete-frame decode and transport
  integrity claims state the excluded trailing bytes explicitly.

## Remaining scope

The validated pipeline uses complete-frame HEVC. Slice output, intra refresh,
alternate modes, rotation, remote ISP control, FC UART, temperature/battery
telemetry, RF power accuracy and flight/range performance remain outside these
results. Mixed-light AWB and broader camera stability also require further work.

See [camera details](../soc/ar9311-p1-camera.md),
[calibration/settings](../firmware/p1-calibration-and-settings.md), and
[boot/recovery](../firmware/p1-boot-and-recovery.md) for the corresponding designs.
