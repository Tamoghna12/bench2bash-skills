# bench2bash-skills

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Latest release](https://img.shields.io/github/v/release/Tamoghna12/bench2bash-skills)](https://github.com/Tamoghna12/bench2bash-skills/releases/latest)
[![Lint](https://github.com/Tamoghna12/bench2bash-skills/actions/workflows/lint.yml/badge.svg)](https://github.com/Tamoghna12/bench2bash-skills/actions/workflows/lint.yml)
[![Contributions welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

**Two Claude AI skills for bioinformatics — one for learning, one for reference.**

`bioinformatics-tutor` teaches biological intuition through Socratic questioning,
adapting to your background. `bioinformatics-fundamentals` keeps Claude grounded
in correct format specifications, tool behaviour, and the silent failures that
break real pipelines.

Built for the [bench2bash](https://tamoghna12.github.io/bench2bash.github.io/)
newsletter by [Tamoghna Das](https://www.linkedin.com/in/tamoghna-das/) ·
PhD Researcher, Loughborough University.

---

## Table of Contents

- [Why these skills exist](#why-these-skills-exist)
- [Skills at a glance](#skills-at-a-glance)
  - [bioinformatics-tutor](#bioinformatics-tutor)
  - [bioinformatics-fundamentals](#bioinformatics-fundamentals)
- [What is covered](#what-is-covered)
- [Quick start](#quick-start)
  - [Install in Claude Code](#install-in-claude-code)
  - [Install from source](#install-from-source)
  - [Use with any AI agent](#use-with-any-ai-agent)
- [Example interactions](#example-interactions)
- [How the skills work](#how-the-skills-work)
- [Repo structure](#repo-structure)
- [Contributing](#contributing)
- [Versioning](#versioning)
- [License](#license)

---

## Why these skills exist

Most bioinformatics errors are not syntax errors. They are:

- **Silent failures** — a pipeline runs cleanly and produces plausible-looking
  output that is biologically wrong (pair-breaking in Hi-C, wrong coordinate
  system, subread BAM used instead of CCS).
- **Conceptual gaps** — a tool is run correctly but the user doesn't know what
  biological question they're actually answering, so they can't tell when the
  result is wrong.
- **Missing context** — a correct answer to "what flag do I use?" that doesn't
  explain why that flag exists, so the user can't adapt when their data is different.

These skills address all three. `bioinformatics-fundamentals` knows the exact
specifications and failure modes. `bioinformatics-tutor` knows how to build the
intuition that prevents the failures from happening in the first place.

---

## Skills at a glance

### bioinformatics-tutor

> *"Teach the biology before the tool. The tool is just the mechanism."*

A Socratic teaching skill for anyone building biological intuition in
bioinformatics — from a wet lab scientist opening a terminal for the first time,
to a software engineer who can write elegant code but has never thought about
what a gene actually does, to a PhD student who knows both sides but hasn't yet
integrated them.

**The core idea — the biological eye:**
Every interaction has two moments that matter. Before computation: frame the
biological question first, not the tool. After computation: validate the result
against what biology predicts — not just whether the pipeline ran without errors.

**How it adapts to you:**

| Your background | What the skill does |
|-----------------|---------------------|
| Wet lab / biology-first | Honours your domain knowledge; maps computational steps to lab procedures you already know |
| CS / data-first | Slows down on biology; prevents the abstraction trap (code that runs perfectly on biologically meaningless data) |
| Early PhD (mixed) | Fills in the reference frame — what does "good" actually look like for your data type? |

**Domains covered:** Genome assembly · Read alignment · Variant calling (SNPs,
indels, SVs) · RNA-seq · ChIP-seq / ATAC-seq · Metagenomics / 16S amplicon ·
Bacterial pangenomics · Genome-scale metabolic modelling · Single-cell RNA-seq
and ATAC-seq

**When to trigger:** Say anything like — *"I'm new to bioinformatics"* ·
*"what does this result mean?"* · *"where do I start?"* · *"I don't understand
why"* · *"does this output make sense?"* · *"I'm a biologist learning to code"*

---

### bioinformatics-fundamentals

> *"The terminal doesn't know biology. You have to."*

A precise technical reference skill for working bioinformaticians. When Claude
has this skill, it knows the correct specifications — not approximations — for
the formats, tools, and failure modes that matter in real genomics pipelines.

**The problem it solves:** Many bioinformatics errors are not caught by tools
because the tools ran correctly on the wrong input. A region filter applied before
a pair filter produces empty output — no error, no warning, just zero reads. This
skill knows these failure modes and the correct ordering of operations.

**Core technical coverage:**

| Category | What's included |
|----------|----------------|
| SAM/BAM | All 12 FLAG bits, common combinations (99, 147, 83, 163…), MAPQ formula, all CIGAR ops, optional tags (NM, MD, AS, SA, HP…) |
| Filtering patterns | Pair-breaking trap and fix, paired-end vs Hi-C vs single-end filtering order, FASTQ extraction sync |
| File formats | AGP (full column spec, gap types, linkage evidence, coordinate validation), FASTQ Phred+33/+64 encoding, BED, FASTA |
| Sequencing tech | PacBio HiFi (CCS vs subread, kinetics tags), Hi-C (ligation junctions, cis/trans ratios, artefact types), Illumina (adapters, error profile), Oxford Nanopore (Dorado basecalling, error profile) |
| Assembly QC | N50/NG50/L50, BUSCO (lineage selection, thresholds), Merqury QV, GenomeScope2, coverage calculations |
| Structural variants | Manta, DELLY, Sniffles2, SURVIVOR multi-caller merge |
| Variant annotation | VEP, SnpEff, gnomAD frequency filtering, CADD/SIFT/PolyPhen-2/GERP |
| Hi-C scaffolding | YAHS, SALSA2, 3D-DNA comparison, Juicebox curation |
| Methylation | Bismark WGBS pipeline, ONT direct methylation (modkit), MethylKit/BSmooth DMR analysis |
| GenomeArk access | VGP S3 directory structure (v1–v3), QC data paths, AWS CLI patterns, path normalisation |
| Genomic analysis | Karyotype curation, telomere BED analysis (tidk), NCBI Taxonomy and Datasets integration |
| Reproducibility | Conda/Mamba, Docker/Singularity/BioContainers, Snakemake, Nextflow/nf-core, SLURM, FAIR data |
| Troubleshooting | Decision trees for empty output, low mapping rate, proper pairs lost, HiFi/Hi-C-specific failures |

**When to trigger:** Any mention of — *samtools* · *minimap2* · *BWA-MEM2* ·
*Dorado* · *BUSCO* · *N50* · *AGP* · *Hi-C filtering* · *empty BAM* ·
*GenomeArk* · *MAPQ* · *structural variants* · *Snakemake* · *Nextflow* ·
*methylation* · or any file format, tool, or QC metric in the table above.

---

## What is covered

The skills together span the full breadth of modern genomics workflows:

```
Experimental data
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│  Raw reads (FASTQ)                                               │
│  ├── PacBio HiFi · Oxford Nanopore · Illumina · Hi-C            │
│  └── Quality control: FastQC, NanoStat, seqkit                  │
└──────────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│  Alignment (SAM/BAM)                                             │
│  ├── minimap2 (map-hifi, map-ont) · BWA-MEM2 · STAR · HISAT2   │
│  ├── Filtering: flags, MAPQ, region, pair-ordering              │
│  └── QC: samtools flagstat/stats/coverage                       │
└──────────────────────────────────────────────────────────────────┘
       │
       ├──────────────────────────────────┬──────────────────────────
       ▼                                  ▼
┌──────────────────┐              ┌──────────────────────────────┐
│  Genome assembly │              │  Read-based analyses         │
│  ├── Hifiasm     │              │  ├── Variant calling         │
│  ├── Hi-C        │              │  │   (GATK, DeepVariant,     │
│  │   scaffolding │              │  │   Manta, Sniffles2)       │
│  │   (YAHS,      │              │  ├── RNA-seq / DE            │
│  │   SALSA2,     │              │  │   (STAR, Salmon, DESeq2)  │
│  │   3D-DNA)     │              │  ├── ChIP-seq / ATAC-seq     │
│  ├── QC: BUSCO,  │              │  │   (MACS2, IDR)            │
│  │   Merqury,    │              │  ├── Metagenomics            │
│  │   GenomeScope │              │  │   (QIIME2, Kraken2,       │
│  └── AGP format  │              │  │   HUMAnN3)                │
└──────────────────┘              │  ├── Single-cell RNA-seq     │
                                  │  │   (Seurat, Scanpy)        │
                                  │  └── Methylation (Bismark,  │
                                  │       modkit)                │
                                  └──────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│  Downstream / specialist analyses                                │
│  ├── Variant annotation: VEP, SnpEff, gnomAD, ClinVar          │
│  ├── Bacterial pangenomics: Panaroo, Roary, ANI                 │
│  ├── Metabolic modelling: CarveMe, COBRApy, FBA                 │
│  ├── Phylogenomics / population genomics                        │
│  └── GenomeArk / VGP data access                                │
└──────────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│  Reproducibility                                                  │
│  ├── Conda/Mamba · Docker/Singularity · BioContainers           │
│  ├── Snakemake · Nextflow · nf-core                             │
│  └── FAIR data · SLURM/HPC · version control                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Quick start

### Install in Claude Code

**From the latest release (recommended):**

```bash
# Download and install both skills
curl -LO https://github.com/Tamoghna12/bench2bash-skills/releases/latest/download/bioinformatics-tutor.skill
curl -LO https://github.com/Tamoghna12/bench2bash-skills/releases/latest/download/bioinformatics-fundamentals.skill

claude skill install bioinformatics-tutor.skill
claude skill install bioinformatics-fundamentals.skill
```

Or drag either `.skill` file into the Claude Code window.

**In Claude.ai / Claude Cowork:** drag the `.skill` file into the Skills panel.

### Install from source

```bash
git clone https://github.com/Tamoghna12/bench2bash-skills.git
cd bench2bash-skills

claude skill install bioinformatics-tutor/
claude skill install bioinformatics-fundamentals/
```

### Use with any AI agent

The skills are plain Markdown. No Claude-specific runtime is required — any
agent or LLM that can read files can use them.

**Option A — paste the skill into your system prompt:**

```bash
cat bioinformatics-tutor/SKILL.md
cat bioinformatics-fundamentals/SKILL.md
```

**Option B — reference raw GitHub URLs in your agent configuration:**

```
# Teaching / intuition building
https://raw.githubusercontent.com/Tamoghna12/bench2bash-skills/main/bioinformatics-tutor/SKILL.md

# Technical reference
https://raw.githubusercontent.com/Tamoghna12/bench2bash-skills/main/bioinformatics-fundamentals/SKILL.md
```

**Option C — load specific reference files for deeper context:**

Each `SKILL.md` describes which `references/` file to load for each topic.
For example, if you need the full SAM/BAM specification, load
`bioinformatics-fundamentals/references/reference.md` directly.

---

## Example interactions

### Teaching: wet lab biologist learning RNA-seq alignment

> **User:** My supervisor told me to "align my RNA-seq reads to the genome".
> I've never done this before and I'm not sure what it means.
>
> **Claude (with tutor skill):**
> Before we touch any tool — what do you think RNA-seq reads actually *are*,
> physically?
>
> Each read in your FASTQ file is a short sequence from a fragment of an mRNA
> molecule that was inside your cells at the exact moment you froze the sample.
> So when we say "align to the genome", we're asking one question per read:
> *which gene in the genome produced this mRNA fragment?*
>
> Think of the genome as the master recipe book. Your RNA-seq reads are torn-out
> fragments from pages in that book — and alignment is the process of figuring
> out which page each fragment came from.
>
> Once we've matched each read to its gene, what do you think counting the reads
> per gene tells us about your experiment?

---

### Teaching: PhD student debugging empty Hi-C output

> **User:** I ran `samtools view -L targets.bed input.bam | bamtools filter
> -isProperPair true` and now I have zero reads. The region definitely has reads.
>
> **Claude (with tutor skill):**
> This is one of the most instructive failures in Hi-C work — let me ask you
> something first. What does a Hi-C read pair actually look like in terms of
> where the two reads land in the genome?
>
> Unlike standard paired-end sequencing, Hi-C mates *intentionally* map to
> distant locations — often different chromosomes. That inter-chromosomal
> contact is the signal you're measuring: chromatin folded in 3D space.
>
> Now trace what `-L targets.bed` does. It checks each read in isolation and
> has no concept of mates. If R1 maps into your region but R2 maps to
> chromosome 7, the region filter keeps R1 and silently discards R2. Now R1
> has no mate, so it fails `isProperPair`. Every read disappears.
>
> The fix is to reverse the order — pair filter first, region filter second.
> Can you reason through why that ordering works where this one doesn't?

---

### Reference: decoding a SAM flag

> **User:** I see FLAG 2064 in my long-read BAM. What is it?
>
> **Claude (with fundamentals skill):**
> FLAG 2064 = 2048 + 16:
>
> - **2048** (0x800): supplementary alignment — this read maps to two locations;
>   the primary alignment is elsewhere in the BAM, and this is the secondary
>   mapping half of a split-read event.
> - **16** (0x010): the read is on the reverse strand at this alignment.
>
> In long-read data (HiFi, ONT), split reads are common at structural variant
> breakpoints — one read spans the junction, mapping partly to one locus and
> partly to another. The primary alignment has an `SA` tag containing the
> coordinates of this supplementary alignment.
>
> To exclude supplementaries: `samtools view -F 2048`

---

### Reference: diagnosing empty BAM after filtering

> **User:** `samtools view -b -L myregion.bed input.bam | bamtools filter
> -isProperPair true` gives me an empty BAM. The region has reads.
>
> **Claude (with fundamentals skill):**
> Classic pair-breaking trap. `-L` filters each read independently — when
> mates map to distant loci (Hi-C by design; paired-end data occasionally),
> R2 is outside the region and gets dropped. R1 survives but is now orphaned,
> so it fails `isProperPair`. Everything is gone.
>
> Fix — reverse the order:
>
> ```bash
> samtools view -b -f 2 -L myregion.bed input.bam > output.bam
> ```
>
> `-f 2` (proper pairs) evaluates while both mates are still present, then
> `-L` restricts to the region.
>
> **Diagnose first:**
>
> ```bash
> # Are there any reads in the region at all?
> samtools view -c input.bam chr1:1000000-2000000
>
> # Where do their mates map?
> samtools view input.bam chr1:1000000-2000000 \
>   | awk '{print $7}' | sort | uniq -c | sort -rn | head
> ```

---

### Reference: ONT basecalling model selection

> **User:** Should I use `hac` or `sup` in Dorado? What is the actual difference?
>
> **Claude (with fundamentals skill):**
> Both use the same neural network architecture, but `sup` (super-accuracy)
> uses a larger model with more parameters.
>
> - `hac` (high-accuracy): ~Q20–Q25 typical accuracy; 3–5× faster than `sup`;
>   good for assembly and most genomic analyses.
> - `sup` (super-accuracy): ~Q25–Q30; the right choice when you need the highest
>   per-base accuracy — variant calling, rare variant detection, methylation
>   calling where base-level errors matter.
>
> ```bash
> # hac — standard for most workflows
> dorado basecaller hac pod5_directory/ > reads.bam
>
> # sup — publication-quality variant calling
> dorado basecaller sup pod5_directory/ > reads.bam
>
> # sup + 5mCpG methylation in one pass
> dorado basecaller sup,5mCG_5hmCG pod5_directory/ > reads_methyl.bam
> ```
>
> Always match the model to the flow cell chemistry: R9.4.1 and R10.4.1
> have separate model sets. Using a mismatched model silently degrades accuracy.

---

## How the skills work

### Architecture

Each skill is a directory with two components:

```
skill-name/
├── SKILL.md          # Instructions for the AI: what to do, how to behave,
│                     # which reference file to consult for which topic
└── references/
    ├── file1.md      # Deep technical content the AI reads on demand
    └── file2.md      # Structured around the topics SKILL.md points to
```

`SKILL.md` is always loaded. The `references/` files are loaded when the
topic arises — they are deliberately separate so that a skill remains
lightweight when the full reference depth isn't needed.

### The `.skill` format

`.skill` files are ZIP archives of the skill directory. They are the
standard distribution format for Claude skills and can be installed
with a single drag-and-drop or CLI command.

### Skill interaction

The two skills are designed to complement each other:

- **tutor** answers *"why does this exist?"* and *"what should I expect?"*
- **fundamentals** answers *"what is the exact specification?"* and *"what went wrong?"*

When someone is confused about why a format or filtering rule exists,
the tutor skill provides the conceptual grounding. When someone needs
the precise flag value or tool command, fundamentals provides it.

---

## Repo structure

```
bench2bash-skills/
│
├── bioinformatics-tutor/
│   ├── SKILL.md                          # Pedagogy, Socratic method, learner-type detection
│   └── references/
│       ├── concepts.md                   # Biological story + validation benchmarks (9 domains)
│       ├── learning-paths.md             # Stage-by-stage progressions by learner background
│       └── single-cell.md               # scRNA-seq / scATAC-seq full teaching reference
│
├── bioinformatics-fundamentals/
│   ├── SKILL.md                          # Format specs, filtering patterns, QC
│   └── references/
│       ├── reference.md                  # SAM/BAM spec, AGP, FASTQ, tools, ONT, SVs, methylation
│       ├── common-issues.md              # Troubleshooting guide with diagnostic decision trees
│       ├── genomeark-data-access.md      # VGP GenomeArk S3 paths, QC locations (v1–v3)
│       ├── genomic-analysis-patterns.md  # Karyotype, telomere analysis, NCBI integration
│       └── workflow-reproducibility.md   # Conda, Snakemake, Nextflow, containers, SLURM, FAIR
│
├── releases/
│   ├── bioinformatics-tutor.skill        # Packaged ZIP for one-click install
│   └── bioinformatics-fundamentals.skill
│
├── .github/
│   ├── ISSUE_TEMPLATE/                   # Structured templates: skill-gap, bug, feature
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── workflows/
│       ├── lint.yml                      # Markdown lint + SKILL.md frontmatter validation
│       └── release.yml                   # Auto-package and publish on version tag
│
├── CHANGELOG.md                          # Keep a Changelog format
├── CONTRIBUTING.md                       # Style guide, workflow, how to regenerate .skill files
├── CODE_OF_CONDUCT.md                    # Contributor Covenant 2.1
├── SECURITY.md
├── LICENSE                               # MIT
└── .markdownlint.yml
```

---

## Contributing

Contributions are welcome. The most valuable contributions are:

- **Factual corrections** — wrong commands, outdated tool behaviour, incorrect benchmarks
- **Coverage gaps** — domains that are missing or too shallow
- **New skills** — a SKILL.md for a domain not yet covered

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full workflow, style guide,
and how to regenerate `.skill` files after making changes.

For bugs and gaps, please [open an issue](https://github.com/Tamoghna12/bench2bash-skills/issues/new/choose)
using the appropriate template before submitting a PR.

---

## Versioning

This project follows [Semantic Versioning](https://semver.org/):

- **MAJOR** — breaking changes to SKILL.md structure or frontmatter format
- **MINOR** — new skills, new reference files, significant new sections
- **PATCH** — corrections, clarifications, minor additions

All notable changes are documented in [CHANGELOG.md](CHANGELOG.md).
Releases are tagged in git (`v1.0.0`, `v1.1.0`, …) and published on the
[GitHub Releases](https://github.com/Tamoghna12/bench2bash-skills/releases) page
with the packaged `.skill` files attached.

---

## License

MIT — see [LICENSE](LICENSE).

---

## About bench2bash

[bench2bash](https://tamoghna12.github.io/bench2bash.github.io/) is a newsletter
and learning resource for biologists learning computational tools — practical
terminal and AI skills for life scientists, every Tuesday.

[Tamoghna Das](https://www.linkedin.com/in/tamoghna-das/) ·
PhD Researcher, Loughborough University
