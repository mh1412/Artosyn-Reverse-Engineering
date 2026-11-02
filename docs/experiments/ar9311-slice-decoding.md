# AR9311 slice and incomplete-picture decoding

[Experiments](README.md) · [AR9311](../soc/ar9311.md)

Scope: Gnd485 AR9311 / Wave521C media experiments consolidated on 2026-10-26,
using the original Artosyn codec firmware and a custom Linux host driver.

## Question

Can the decoder expose correct regions of a picture before all its slices arrive,
then accept the remaining slices and finish that same picture? Such behavior
could support lower latency, but decoding, region completion and display safety
must be established separately.

## Setup and controls

The investigation used H.264 and H.265 pictures with controlled slice boundaries,
known visual patterns and host-decoded reference pixels. Progressive-input tests
withheld later data while observing decoder output. Separate prefix tests supplied
one, two or three of four slices and explicitly finalized the incomplete picture.

Firmware/parser probes and temporary driver changes were part of the investigation;
these results do not describe an unmodified general-purpose decoder API.

## Results

| Test | Observation |
| --- | --- |
| Ordinary full-frame decoding | Hardware H.264/H.265 output matched the tested reference pictures |
| Progressive picture input | Complete pictures decoded correctly, without demonstrated early usable output |
| Forced finalization after 1/2/3 of 4 slices | Six controls across both codecs produced exact supplied color bands |
| Missing regions after forced completion | Matched previous-frame content in the tested controls |
| Additional progressive-input runs | 18 exact completed frames across six runs; no early output-memory changes observed |
| Diagnostic codec firmware patch | A two-byte RAM patch exposed common wait-routine callers while preserving tested decoding |

## Interpretation

The hardware can reconstruct supplied slices when firmware is told to finish an
incomplete picture. That result does not establish resumable decoding: forced
completion ends the picture rather than proving it can remain open, publish
finished regions and accept later slices into the same output.

No trustworthy decoded-row/slice-ready signal was identified. Safe early scan-out
and a reduction in glass-to-glass latency were not demonstrated. An open host
driver and a successful diagnostic firmware patch also do not establish a complete
codec CPU ISA or a source-buildable firmware replacement.

## What would establish the next step

A positive resumable result needs correct, non-stale pixels while later slices
are withheld; a reliable completion indication for those regions; correct final
output after the remaining data arrives; and repeatability across patterns and
reference pictures. Display tests would then need synchronization with ongoing
device writes and an independent latency measurement.

The experiments make firmware scheduling and output handling useful investigation
targets. They do not prove that hardware continuation, state retention or memory
writeback imposes no further constraints.
