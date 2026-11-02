# Project structure

[Documentation index](../README.md) · [Documentation conventions](../CONTRIBUTING.md)

The root README is the wiki's front page. Section READMEs group articles by chip
or device family; device pages connect hardware, firmware, protocol and experiment
studies for a particular product. Ordinary relative Markdown links work on GitHub
without a site generator.

## Layout

```text
README.md                   Front page and documentation index
CONTRIBUTING.md              Scope, evidence and writing conventions
docs/
├── structure.md             Organization and page ownership
├── research-status.md       Coverage, corrections and open questions
├── devices/                 Products, boards and teardowns
├── soc/                     Silicon, registers and companion chips
├── firmware/                Images, boot, runtime, updates and recovery
├── protocols/               Transport and message formats
├── systems/                 Service ownership and end-to-end media paths
├── experiments/             Procedures, observations and validation limits
└── references/              Public sources and analysis methods
assets/                     Supporting images and small evidence artifacts
```

Each topic directory has a `README.md` index. Start a subject as one article;
split it only when the parts answer distinct questions. Avoid empty placeholders
and standalone pages that repeat an overview.

## Where a finding belongs

| Section | Owns | Example |
| --- | --- | --- |
| Devices | Physical product identity, components and wiring | Gnd485 radio reset pin |
| SoC | Shared silicon behavior | AR9311 GPIO register model |
| Firmware | Software images and binary behavior | Boot selection or vendor radio workers |
| Protocols | Information crossing a named boundary | SDIO framing or video packet headers |
| Systems | How components work together | Camera-to-display path and process ownership |
| Experiments | A dated investigation and its evidence | GPIO readback or incomplete-picture decode test |
| References | Public sources and interpretation methods | SDK history or synthetic ELF assumptions |

Keep each specification in one authoritative page and link to it from other
views. An overview connects findings; an experiment preserves the procedure and
conditions that support them. Teardowns belong with the device. Companion chips
belong under `soc/` when they are part of an Artosyn-based system.

## Navigation and names

- Use lowercase, hyphenated filenames. Retain product, board and firmware
  identifiers in the text; do not silently equate aliases or revisions.
- Link each article from its section index and provide a route back. Group
  overview pages before detailed studies, then link related evidence.
- Preserve existing article paths where practical. When consolidating pages,
  update incoming links and retain unique findings and evidence limits.
- Use relative links for documentation and assets. Keep pages self-contained;
  no reader should need the author's workstation or private research files.

The [documentation conventions](../CONTRIBUTING.md) define stock-study scope,
evidence labels and source handling. The [coverage page](research-status.md)
tracks unresolved research separately from the navigation.
