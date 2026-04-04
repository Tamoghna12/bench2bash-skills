---
name: bioinformatics-fundamentals
description: >
  Foundation knowledge for genomics and bioinformatics workflows. Use this skill
  whenever working with sequencing data (PacBio HiFi, Hi-C, Illumina), SAM/BAM
  files, AGP assembly formats, FASTQ/FASTA, or BED files. Trigger for: debugging
  alignment or filtering issues, paired-end vs single-end data questions, MAPQ or
  PHRED score interpretation, empty output troubleshooting, broken read pairs,
  genome assembly QC (BUSCO, Merqury, GenomeScope), AGP coordinate validation,
  unloc assignments, GenomeArk data access, karyotype curation, chromosome count
  analysis, and any general bioinformatics data analysis. Also trigger when the
  user mentions samtools, bamtools, minimap2, BWA-MEM2, or any of the common
  bioinformatics file-processing patterns described in this skill — even if they
  don't explicitly frame it as a "fundamentals" question.
---

# Bioinformatics Fundamentals

This skill provides essential reference knowledge for genomics and bioinformatics
workflows. Use it to ground your answers in correct format specifications,
filtering semantics, and tool behaviour — especially for paired-end data, where
subtle ordering mistakes cause silent failures.

Supporting files live in `references/` — read them when you need depth beyond
what is summarised here:
- `reference.md` — Complete SAM/BAM spec, CIGAR ops, AGP format, FASTQ encoding,
  tool command reference, sequencing tech specs, assembly quality metrics
- `common-issues.md` — Troubleshooting guide (empty outputs, pair-breaking,
  Hi-C and HiFi specific issues, AGP errors, diagnostic commands)
- `genomeark-data-access.md` — GenomeArk AWS S3 directory structure, QC data
  locations (GenomeScope/BUSCO/Merqury), path normalisation, assembly date
  extraction
- `genomic-analysis-patterns.md` — Karyotype curation, haploid vs diploid
  chromosome counts, phylogenetic tree species mapping, BED/telomere analysis,
  NCBI data integration

---

## SAM/BAM Essentials

### SAM Flags (bitwise — flags are additive)

| Flag (hex) | Decimal | Meaning |
|---|---|---|
| 0x0001 | 1 | Read is paired |
| 0x0002 | 2 | Proper pair |
| 0x0004 | 4 | Read unmapped |
| 0x0008 | 8 | Mate unmapped |
| 0x0010 | 16 | Read on reverse strand |
| 0x0020 | 32 | Mate on reverse strand |
| 0x0040 | 64 | First in pair (R1) |
| 0x0080 | 128 | Second in pair (R2) |
| 0x0100 | 256 | Secondary alignment |
| 0x0400 | 1024 | PCR/optical duplicate |
| 0x0800 | 2048 | Supplementary alignment |

Common combinations: properly-paired R1 = **99** (1+2+32+64); properly-paired
R2 = **147** (1+2+16+128).

### MAPQ

`MAPQ = -10 × log₁₀(P(wrong mapping))`

- ≥ 60 → very high confidence (< 0.0001 % error)
- ≥ 30 → good (< 0.1 %)
- ≥ 20 → acceptable (< 1 %)
- = 0 → multi-mapper **or** unmapped — distinguish with flag 0x0004

### CIGAR String

`M` match/mismatch · `I` insertion · `D` deletion · `S` soft-clip ·
`H` hard-clip · `N` skipped region (RNA-seq splicing)

Example: `50M5I45M` = 50 bp match, 5 bp insertion, 45 bp match.

See `references/reference.md` for the full SAM mandatory-fields table, optional
tags (NM, MD, AS, XS, SA, HP …), and complete CIGAR operation semantics.

---

## Sequencing Technologies

### PacBio HiFi
- Read length: 10–25 kb typical · accuracy > 99.9 % (Q20+)
- Circular Consensus Sequencing — effectively **single-end**
- Best mappers: `minimap2 -x map-hifi` (or `map-pb`)
- Use cases: de novo assembly, SV detection, Iso-Seq, haplotype phasing

### Hi-C (Chromatin Conformation Capture)
- Short paired-end reads (100–150 bp); pairs **intentionally map to distant loci**
- R1 and R2 routinely land on different scaffolds/chromosomes — this is normal
- Best mappers: BWA-MEM2 or BWA-MEM in paired-end mode
- **Critical:** region-based filtering easily breaks pairs (see Patterns below)
- Use cases: genome scaffolding, 3D chromatin structure, phasing, assembly QC

### Illumina Short Reads
- 50–300 bp; paired-end or single-end; high throughput
- Best mappers: BWA-MEM2, BWA-MEM, Bowtie2 (local), STAR (RNA-seq)

See `references/reference.md` for detailed error profiles and quality metrics
per technology.

---

## Common Tools

### `samtools view` — filter / convert SAM/BAM

Key flags: `-b` BAM out · `-h` include header · `-f INT` require flags ·
`-F INT` exclude flags · `-q INT` min MAPQ · `-L FILE` keep reads in BED regions

> **Important:** `-L` filters each read individually — it does not know about
> mates. If one mate maps outside the region, the pair is silently broken and
> the proper-pair flag drops.

### `bamtools filter`
`isProperPair: true` applies stricter pair-validity checks than samtools `-f 2`.

### `samtools fastx`
Converts BAM → FASTQ/FASTA. Use `--i1-flags 2` to require proper pairs so that
R1 and R2 output files stay in sync.

See `references/reference.md` for full command references with all options.

---

## Critical Patterns

### Pattern 1: Filtering paired-end data by genomic region

The most common silent failure: applying `-L` before ensuring proper pairs.

```bash
# ❌ WRONG — breaks pairs when mates map to different regions
samtools view -b -L regions.bed input.bam \
  | bamtools filter -isProperPair true
# Result: empty output (all pairs broken before the pair check)

# ✅ CORRECT — pair filter first, then region filter
samtools view -b -f 2 -L regions.bed input.bam > output.bam
```

Why this matters: with Hi-C data especially, mate pairs routinely map to
different chromosomes. The region filter sees only one read of the pair, marks
it as non-proper, and the downstream pair filter removes everything.

### Pattern 2: FASTQ extraction from filtered BAM

```bash
# Paired-end (keep pairs in sync)
samtools fastx -1 R1.fq.gz -2 R2.fq.gz --i1-flags 2 input.bam

# Single-end
samtools fastx -0 output.fq.gz input.bam
```

### Pattern 3: Quality filtering

```bash
# Conservative (high-confidence variants, assemblies)
samtools view -b -q 30 -f 2 -F 256 -F 2048 input.bam
# MAPQ ≥ 30, proper pairs, no secondary, no supplementary

# Permissive (low-coverage or Hi-C data)
samtools view -b -q 10 -F 4 input.bam
# MAPQ ≥ 10, mapped only
```

---

## Common Issues — Quick Reference

| Symptom | Likely cause | Fix |
|---|---|---|
| Empty output after region filter | Pair-breaking: region filter before pair filter | Reverse order — pair filter first |
| R1 / R2 read count mismatch | Filtering broke some pairs | Add `--i1-flags 2` to `samtools fastx` |
| Low Hi-C mapping rate | Normal — chimeric reads are expected | Use Hi-C-specific pipelines (HiC-Pro, Juicer) |
| Proper pairs lost after mapping | Insert size, reference mismatch, wrong orientation flags | Check with `samtools stats` and `samtools flagstat` |

For full diagnostic commands, Hi-C/HiFi-specific issues, and AGP processing
errors see `references/common-issues.md`.

---

## File Formats — Quick Reference

**FASTA:** `>id description\nSEQUENCE` — no quality scores.

**FASTQ:** 4 lines per read: `@id / SEQUENCE / + / QUALITY` (Phred+33).

**BED:** tab-delimited, **0-based half-open** `[start, end)`. Columns: chrom,
chromStart, chromEnd, name, score, strand.

**AGP:** tab-delimited genome assembly format, **1-based closed** `[start, end]`.
Object and component lengths must satisfy:
`obj_end − obj_beg + 1 == comp_end − comp_beg + 1`

**SAM/BAM:** 1-based coordinates. Convert to BED: `BED_start = SAM_start − 1`.

See `references/reference.md` for complete AGP column specification, gap types,
and validation rules.

---

## Quality Metrics

**N50:** Length of the shortest contig at which ≥ 50 % of the total assembly
is contained in contigs of that length or longer. L50 = the count of contigs
needed to reach N50. NG50 normalises against expected genome size — prefer it
for cross-species comparisons.

**Coverage / Depth:** Coverage = % of bases covered by ≥ 1 read. Depth =
mean reads per base.

Recommended depths: HiFi assembly 30–50×; variant calling ≥ 30×;
Hi-C scaffolding 50–100× genomic coverage.

---

## Coordinate Systems

| System | First base | Interval notation | Used in |
|---|---|---|---|
| 1-based closed | 1 | [2, 5] includes 2,3,4,5 | SAM, VCF, GFF, AGP |
| 0-based half-open | 0 | [2, 5) includes 2,3,4 | BED, BAM binary |

Conversion: `BED_start = SAM_start − 1`; `BED_end = SAM_end`.

---

## Best Practices

1. **Know your data type first** — paired vs single-end determines every
   downstream filtering decision.
2. **Filter in the right order** — proper-pair flag *before* region filters,
   always.
3. **Hi-C is not DNA-seq** — distant mate pairs are the signal, not an error.
   Don't over-filter on MAPQ; use Hi-C-aware pipelines.
4. **HiFi is single-end** — no pair management needed; use `map-hifi` preset.
5. **Validate outputs** — `samtools flagstat` and `samtools stats` after every
   major filtering step; check file size and read counts.
6. **Test datasets should be self-contained** — when creating subsets for
   testing, make sure both mates of every pair land inside the region.

---

## Companion Skill

- `bioinformatics-tutor` — If someone is confused about *why* a format or
  filtering rule exists (not just what it is), hand off to the tutor skill.
  Fundamentals answers "what" and "how"; tutor answers "why" and "does this
  make biological sense".
