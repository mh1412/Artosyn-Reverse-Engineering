# Project structure

[Documentation index](../README.md)

## Purpose

This repository is a GitHub-browsable Markdown wiki for Artosyn reverse-engineering
knowledge. The root README is its front page. Section READMEs provide navigation;
individual pages hold findings and link to the experiments that support them.
Ordinary relative Markdown links are sufficient; a site generator is optional.

## Layout

Articles currently live directly under their topic directories. This abbreviated
tree shows existing pages; each section README links to the full set.

```text
artosyn-re/
├── README.md                         # Front page and documentation index
├── CONTRIBUTING.md                   # Evidence and writing conventions
├── docs/
│   ├── structure.md                  # Organization and research workflow
│   ├── research-status.md            # Coverage, corrections and open questions
│   ├── systems/
│   │   ├── README.md
│   │   ├── video-paths.md             # End-to-end architecture
│   │   └── ascent-groundstation.md    # Application study
│   ├── devices/
│   │   ├── README.md
│   │   ├── ascent-vrx.md              # Product, board and interfaces
│   │   └── skydroid-g11.md
│   ├── soc/
│   │   ├── README.md
│   │   ├── ar9311.md                  # Application SoC
│   │   └── ar8030.md                  # Radio/baseband
│   ├── firmware/
│   │   ├── README.md
│   │   └── p1-boot-and-recovery.md    # Board-specific boot and recovery
│   ├── protocols/
│   │   ├── README.md
│   │   ├── ar8030-host-link.md        # Host/radio boundary
│   │   └── ar8030-wireless-format.md  # Over-air framing
│   ├── experiments/
│   │   ├── README.md
│   │   └── open8030-validation.md     # Results and validation limits
│   └── references/
│       ├── README.md
│       └── edgetx-integration.md      # Pinned external assumptions
└── assets/                           # Supporting images and other small assets
```

Start a subject as one page. Split it into a directory when the content justifies
separate pages. Avoid empty placeholders and links to files that do not exist.

## Where a finding belongs

| Section | Owns | Example |
| --- | --- | --- |
| Systems | How components work together and what an application needs | Camera-to-display path or USB ground station |
| Devices | A physical product or board revision | Gnd485 wiring, Sky482 flash package, G11 connectors |
| SoC | Shared silicon behavior | AR9311 GPIO registers or AR8030 execution architecture |
| Firmware | Software images, boot/runtime behavior and implementations | ASW/OTRA containers, boot selection, Open8030 |
| Protocols | Information exchanged across a named boundary | SDIO framing, radio MAC messages, video packet headers |
| Experiments | A dated investigation and its reproducible evidence | GPIO readback audit or incomplete-picture decode test |
| References | Public supporting material | Datasheets, SDKs, papers and upstream repositories |

Keep the authoritative explanation in one place and link to it from other views.
For example, a Gnd485 device page owns its radio reset pin; an AR9311 page owns
the GPIO register model; a protocol page owns the host message format. A system
page connects all three without copying their specifications.

Teardowns live with the device. Both application processors and companion chips
fit under `soc/`; the section title is “SoC and chip studies.” Keep SKY/GND and
USB/SDIO differences explicit even when they share code or hardware families.

## Navigation and names

- Use lowercase, hyphenated names, such as `ascent-vrx`, `ar9311`, and
  `ar8030-host-link`. Keep the original product and board identifiers in the text.
- Link from the root index to each section, from sections to articles, and from
  articles to related studies and evidence. Include a route back to the index.
- Use repository-relative links for pages and assets. Articles must be readable
  without access to the author's workstation or another private repository.
- Keep unresolved identifiers explicit. Do not silently equate product aliases,
  board names or firmware build names.
- Keep implementation repositories responsible for their build/deployment
  instructions. This wiki owns the reusable explanation and research evidence.

## Consolidating existing research

Include hidden research directories within each project and its nested component
projects in discovery. Read experimental reports and deployment records alongside
Markdown narratives; a top-level README may predate the decisive result. Keep
the source-file inventory private and publish self-contained findings here.

In particular, inspect each project's `.local` directory and nested component
`.local` directories. Include Markdown notes, result logs, JSON receipts and
supporting captures. Compare archived checkpoints with later corrections;
copied source trees are not independent experiments. Keep discovery, partial
reading and completed incorporation distinct when assessing coverage.

1. Identify the finding, its research date and relevant board/firmware identifiers.
2. Preserve the technical evidence, binary hashes, tested conditions and limits
   needed to understand or reproduce it. Use the content of existing research
   directly; do not publish a catalog of local source filenames.
3. Read linked corrections and later results. Resolve differences by evidence
   and scope, not by filename or modification time alone.
4. Write the durable finding in its topic page. Preserve the procedure and
   historical sequence in an experiment page where they matter.
5. Bring over needed supporting assets with provenance and repair relative links.
   Reference large/private artifacts by manifest rather than copying whole workspaces.
6. Update the topic index and link related studies and experiments.

Vendored libraries, repeated release trees, tool environments and implementation
READMEs are not automatically wiki articles. Extract the useful explanation,
reconcile historical stages, and write a self-contained account of the findings.
