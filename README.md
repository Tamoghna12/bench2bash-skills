# bench2bash-skills

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Latest release](https://img.shields.io/github/v/release/Tamoghna12/bench2bash-skills)](https://github.com/Tamoghna12/bench2bash-skills/releases/latest)
[![Lint](https://github.com/Tamoghna12/bench2bash-skills/actions/workflows/lint.yml/badge.svg)](https://github.com/Tamoghna12/bench2bash-skills/actions/workflows/lint.yml)
[![Contributions welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

Two Claude skills for bioinformatics — one for learning, one for reference.
Built for [bench2bash](https://tamoghna12.github.io/bench2bash.github.io/) by Tamoghna Das.

---

## Skills

### `bioinformatics-tutor`

A Socratic teaching skill for developing biological intuition. For anyone learning
bioinformatics — whether you're coming from the wet lab, from software engineering,
or from an early PhD trying to integrate both.

**What it does:**
- Detects your background (wet lab / CS / early PhD) and adapts how it teaches
- Explains *why* before *how* — biology first, tools second
- Teaches the "biological eye": frame the question biologically before you touch a
  tool; validate results biologically before you trust them
- Uses Socratic questions to help you reason through problems yourself
- Covers: RNA-seq, genome assembly, variant calling, ChIP-seq, metagenomics,
  bacterial pangenomics, metabolic modelling

**Trigger phrases:**
`"I'm new to bioinformatics"` · `"what does this result mean?"` · `"where do I start?"` ·
`"I don't understand why"` · `"does my output make biological sense?"`

---

### `bioinformatics-fundamentals`

A precise technical reference skill for working bioinformaticians. Keeps Claude
grounded in correct format specifications, filtering semantics, and tool behaviour —
especially for the silent failures that are hard to debug.

**What it does:**
- All 12 SAM FLAG bits decoded; common flag combinations (99, 147, 83, 163 …)
- MAPQ interpretation and CIGAR string semantics
- The pair-breaking trap: why `-L regions.bed | bamtools filter -isProperPair true`
  produces empty output for Hi-C data, and the correct filtering order
- PacBio HiFi vs Hi-C vs Illumina — technology-specific alignment guidance
- Full AGP format specification (columns, gap types, linkage evidence, coordinate validation)
- Assembly QC: N50/NG50/L50, BUSCO lineage selection, Merqury QV, GenomeScope2
- GenomeArk (VGP) AWS S3 access patterns for all project phases (v1–v3)
- Karyotype curation, telomere BED analysis, NCBI Datasets integration

**Trigger phrases:**
`"what flag do I use"` · `"why is my output empty"` · `"MAPQ threshold"` ·
`"samtools view"` · `"Hi-C filtering"` · `"AGP format"` · `"GenomeArk"` · `"BUSCO"` · `"N50"`

---

## Install

### Option 1 — `.skill` file (easiest, one click)

1. Download from [`releases/`](releases/):
   - [`bioinformatics-tutor.skill`](releases/bioinformatics-tutor.skill)
   - [`bioinformatics-fundamentals.skill`](releases/bioinformatics-fundamentals.skill)
2. **Claude Code:** drag the file into the Claude Code window, or run:
   ```bash
   claude skill install bioinformatics-tutor.skill
   claude skill install bioinformatics-fundamentals.skill
   ```
3. **Claude Cowork / Claude.ai:** drag the `.skill` file into the Skills panel.

### Option 2 — Install from source

```bash
git clone https://github.com/Tamoghna12/bench2bash-skills.git
cd bench2bash-skills

claude skill install bioinformatics-tutor/
claude skill install bioinformatics-fundamentals/
```

---

## Example interactions

### bioinformatics-tutor

**Example 1 — Wet lab biologist just starting out**

> **User:** I have RNA-seq data from my experiment and my supervisor told me to
> "align it to the reference genome". I don't really know what that means.
>
> **Claude (with tutor skill):** Before we open any tool — what do you think
> RNA-seq is actually capturing? Each read in your data is a fragment of...
> what, physically? [Pause for answer, or continue:]
>
> Right — a fragment of an mRNA molecule that was in your cells at the moment
> you froze the sample. So when we "align to the genome", we're asking: *which
> gene did each of those mRNA fragments come from?* We're matching the RNA
> sequence back to the place in the genome that produced it.
>
> Think of the genome as a master template, and your RNA-seq reads as
> photocopies of small sections. Alignment is the process of figuring out
> where each photocopy came from in the original template.
>
> Once we've done that alignment, what do you think counting reads per gene
> tells us?

---

**Example 2 — CS person confused about why BUSCO matters**

> **User:** My BUSCO score is 94.3% complete. Is that good or bad?
>
> **Claude (with tutor skill):** What do you think a BUSCO score is measuring —
> what's the underlying biological question?
>
> BUSCO checks your assembly for ~4,000 genes that evolution has kept intact
> across all mammals (or whichever lineage you chose). They're single-copy in
> most mammals, so if your assembly has them all, intact, in one copy —
> that's strong evidence the assembly captured the full gene repertoire.
>
> 94.3% complete for a mammalian genome is solid — you'd be looking for ≥ 95%
> for a chromosome-level VGP assembly. The question is: what happened to the
> missing 5.7%? Are they fragmented (which means the assembly broke those genes
> across scaffolds) or missing (possibly in a gap or a repeat region)?
>
> What does your BUSCO output show for the F (fragmented) vs M (missing) split?

---

**Example 3 — PhD student getting wrong results**

> **User:** I filtered my Hi-C BAM with `samtools view -L targets.bed` and then
> `bamtools filter -isProperPair true` and now I have zero reads. The region
> definitely has reads.
>
> **Claude (with tutor skill):** This is actually a really instructive failure —
> let me ask you first: what do Hi-C read pairs actually look like in terms of
> where the two reads land in the genome?
>
> Unlike standard paired-end DNA-seq, Hi-C reads *intentionally* come from
> distant loci — often different chromosomes entirely. That's the signal you're
> capturing: 3D contacts in the nucleus.
>
> Now think about what `-L targets.bed` does to each read in isolation. It
> doesn't know about mates. If R1 maps into your region but R2 maps to another
> chromosome, the region filter keeps R1 but silently drops R2. With R2 gone,
> R1 can no longer satisfy "isProperPair" — so the pair filter removes it too.
> Everything disappears.
>
> The fix is to reverse the order: apply the pair filter first, then the region
> filter. Can you reason through why that order works?

---

### bioinformatics-fundamentals

**Example 1 — Decoding a SAM flag**

> **User:** I see reads with FLAG 2064 in my BAM. What does that mean?
>
> **Claude (with fundamentals skill):** FLAG 2064 = 2048 + 16.
> - 2048 (0x800): supplementary alignment — this is a chimeric/split read;
>   the primary alignment is elsewhere in the BAM
> - 16 (0x010): read is on the reverse strand
>
> So: a supplementary alignment on the reverse strand. These appear in long-read
> data (HiFi, ONT) where a single read maps to two distant locations (e.g.
> spanning a structural variant breakpoint). The `SA` tag on the primary
> alignment contains the partner coordinates.
>
> If you want to exclude supplementary alignments: `samtools view -F 2048`

---

**Example 2 — Debugging empty BAM output**

> **User:** `samtools view -b -L myregion.bed input.bam | bamtools filter -isProperPair true`
> gives me an empty BAM. The region has reads — I can see them with `samtools view`.
>
> **Claude (with fundamentals skill):** Classic pair-breaking trap. The region
> filter (`-L`) checks each read independently. For Hi-C or any paired-end data
> where mates map to distant loci, R2 maps outside your region and gets dropped.
> R1 survives the region filter but now has no mate, so it fails `isProperPair`.
> Result: empty.
>
> Fix — reverse the order:
> ```bash
> samtools view -b -f 2 -L myregion.bed input.bam > output.bam
> ```
> The `-f 2` (require proper pairs) runs *before* `-L` drops the mates, so
> pairs are evaluated while both reads are still present.
>
> To confirm this is the cause:
> ```bash
> # How many reads are in the region regardless of pairing?
> samtools view -c input.bam chr1:start-end
> # Where do the mates map?
> samtools view input.bam chr1:start-end | awk '{print $7}' | sort | uniq -c
> ```

---

**Example 3 — AGP coordinate question**

> **User:** My AGP has `object_beg=1000001` and `object_end=1000100` for a gap.
> Is that a 100 bp or 101 bp gap?
>
> **Claude (with fundamentals skill):** 100 bp. AGP uses 1-based closed intervals:
> length = `object_end − object_beg + 1 = 1000100 − 1000001 + 1 = 100`.
>
> This trips people up because it looks like a 101-number range. The +1 corrects
> for the fact that both endpoints are *included* in a closed interval. The same
> rule applies to every non-gap line: the component length extracted must equal
> `component_end − component_beg + 1`.
>
> Quick validation:
> ```bash
> awk '$5 !~ /^[NU]$/ {
>   obj_len = $3 - $2 + 1
>   comp_len = $8 - $7 + 1
>   if (obj_len != comp_len) print "MISMATCH line " NR
> }' assembly.agp
> ```

---

## Use with any AI agent

The skills are plain Markdown — any agent that can read files can use them.
No Claude-specific runtime required.

**Paste SKILL.md directly into your system prompt:**
```bash
cat bioinformatics-tutor/SKILL.md
cat bioinformatics-fundamentals/SKILL.md
```

**Reference the raw GitHub URL in agent instructions:**
```
https://raw.githubusercontent.com/Tamoghna12/bench2bash-skills/main/bioinformatics-tutor/SKILL.md
https://raw.githubusercontent.com/Tamoghna12/bench2bash-skills/main/bioinformatics-fundamentals/SKILL.md
```

Load the `references/` files as additional context when deeper technical detail
is needed. Each SKILL.md points to the right reference file for each topic.

---

## Repo structure

```
bench2bash-skills/
├── bioinformatics-tutor/
│   ├── SKILL.md                          # Main skill: pedagogy, Socratic method, learner types
│   └── references/
│       ├── concepts.md                   # Biological story + validation benchmarks per domain
│       └── learning-paths.md             # Stage-by-stage progressions (wet lab / CS / PhD)
│
├── bioinformatics-fundamentals/
│   ├── SKILL.md                          # Main skill: format specs, filtering patterns, QC
│   └── references/
│       ├── reference.md                  # SAM/BAM spec, AGP, FASTQ encoding, tool commands
│       ├── common-issues.md              # Troubleshooting: pair-breaking, low mapping, Hi-C/HiFi
│       ├── genomeark-data-access.md      # VGP GenomeArk S3 paths, QC data locations
│       └── genomic-analysis-patterns.md  # Karyotype, telomere analysis, NCBI integration
│
├── releases/
│   ├── bioinformatics-tutor.skill        # Packaged for one-click install
│   └── bioinformatics-fundamentals.skill
│
├── CHANGELOG.md
└── LICENSE                               # MIT
```

---

## About bench2bash

[bench2bash](https://tamoghna12.github.io/bench2bash.github.io/) is a newsletter
and resource hub for biologists learning computational tools — practical terminal
and AI skills for life scientists, every Tuesday.

By [Tamoghna Das](https://www.linkedin.com/in/tamoghna-das/) · PhD Researcher,
Loughborough University · MIT License
