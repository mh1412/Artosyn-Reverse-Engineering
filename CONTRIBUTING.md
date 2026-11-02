# Documentation conventions

[Documentation index](README.md) · [Project structure](docs/structure.md)

## Scope and evidence

State which product, board revision, firmware version and experiment a result
describes. Keep the application host, radio/baseband and codec firmware distinct.
Separate stock behavior from behavior under modified software.

Use evidence descriptions consistently:

| Label | Meaning |
| --- | --- |
| Observed | Directly recorded on the named hardware or in a named capture |
| Firmware-derived | Recovered from identified binaries, scripts or device trees |
| Inferred | An interpretation supported by evidence, with alternatives or limits |
| Reported | Attributed to an owner, contributor or external source |
| Unknown | Not established by the available evidence |

Track documentation maturity separately: a topic can be indexed, drafted or
consolidated regardless of how much hardware behavior has been validated.
Bench success, simulated behavior and firmware inspection answer different
questions. For example, output frame cadence does not measure glass-to-glass
latency, and a loaded radio firmware image does not establish RF/video parity.

## Topic page outline

Use the parts that fit the subject:

```text
# Descriptive title
Scope: device / board / firmware / date
Summary of the finding
Evidence and confidence
Technical details
Known differences and limitations
Open questions
Sources and related pages
```

For packet formats, include the layer, transport, direction, endianness, field
sizes, checksums, state transitions and a trace/example when available. Label
unknown fields instead of assigning speculative meanings as facts.

## Experiment page outline

```text
# Experiment title
Date and question
Device, firmware hashes, tools and setup
Baseline and relevant modifications
Procedure
Observations and artifacts
Interpretation and limitations
Restored/final state
Follow-up or superseding experiment
```

Record negative results and corrections. A newer conclusion should link to its
supporting result and explain what changed; retain the scope of the older result.

## Sources and artifacts

- Write self-contained explanations using the research content. Do not reference
  local workspace files or require access to them. Cite public references when
  relevant, and preserve research dates, firmware identifiers and measured evidence.
- Identify binary inputs by filename, size and SHA-256. Record extraction offsets,
  load addresses and tool versions where they affect interpretation.
- Keep diagrams, small evidence excerpts and publishable photos beside their
  documentation through relative links. Retain editable diagram sources.
- Keep local working captures, credentials, device-specific secrets, proprietary
  firmware and analysis databases out of routine documentation commits. Describe
  required artifacts and their hashes when the original cannot be included.
- Preserve attribution for borrowed research and tools. A copied reference
  checkout is not evidence of a new experiment on another product.

## Before adding a page

Check its relative links and image paths, add it to the relevant section index,
and make sure each hardware claim identifies its evidence and limitations.
Documentation edits do not require hardware operations or firmware execution.
