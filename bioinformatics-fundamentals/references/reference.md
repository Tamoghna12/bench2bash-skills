# Bioinformatics Fundamentals — Detailed Reference

Complete technical reference for SAM/BAM format internals, AGP assembly format,
FASTQ encoding, tool commands, sequencing technology specifications, and assembly
quality metrics.

---

## Table of Contents

1. [SAM/BAM Complete Reference](#sambam-complete-reference)
2. [AGP Format Specification](#agp-format-specification)
3. [FASTQ Encoding](#fastq-encoding)
4. [Tool Command Reference](#tool-command-reference)
5. [Sequencing Technology Specifications](#sequencing-technology-specifications)
6. [Assembly Quality Metrics](#assembly-quality-metrics)
7. [Coverage Calculations](#coverage-calculations)
8. [Coordinate Systems — Extended](#coordinate-systems--extended)

---

## SAM/BAM Complete Reference

### Mandatory Fields (in column order)

| Col | Field  | Type   | Description |
|-----|--------|--------|-------------|
| 1   | QNAME  | String | Query name (read name). `*` if unavailable. |
| 2   | FLAG   | Int    | Bitwise flag (see table below). |
| 3   | RNAME  | String | Reference sequence name. `*` if unmapped. |
| 4   | POS    | Int    | 1-based leftmost mapping position. 0 if unmapped. |
| 5   | MAPQ   | Int    | Mapping quality. 255 = not available. |
| 6   | CIGAR  | String | CIGAR string. `*` if unavailable. |
| 7   | RNEXT  | String | Ref name of the mate. `=` if same as RNAME. `*` if unavailable. |
| 8   | PNEXT  | Int    | 1-based position of the mate. 0 if unavailable. |
| 9   | TLEN   | Int    | Observed template length (insert size). Negative for right segment. |
| 10  | SEQ    | String | Segment sequence. `*` if not stored. |
| 11  | QUAL   | String | Phred+33 base qualities. `*` if not stored. |

### SAM FLAG Bits (all 12 bits)

| Bit | Decimal | Hex    | Meaning |
|-----|---------|--------|---------|
| 0   | 1       | 0x001  | Read paired |
| 1   | 2       | 0x002  | Properly paired |
| 2   | 4       | 0x004  | Read unmapped |
| 3   | 8       | 0x008  | Mate unmapped |
| 4   | 16      | 0x010  | Read on reverse strand |
| 5   | 32      | 0x020  | Mate on reverse strand |
| 6   | 64      | 0x040  | First in pair (R1) |
| 7   | 128     | 0x080  | Second in pair (R2) |
| 8   | 256     | 0x100  | Secondary alignment |
| 9   | 512     | 0x200  | Read fails platform/vendor QC |
| 10  | 1024    | 0x400  | PCR or optical duplicate |
| 11  | 2048    | 0x800  | Supplementary alignment (chimeric) |

**Common flag combinations:**

| Value | Meaning |
|-------|---------|
| 99    | R1, properly paired, mate on reverse strand (1+2+32+64) |
| 147   | R2, properly paired, read on reverse strand (1+2+16+128) |
| 83    | R1, properly paired, read on reverse strand (1+2+16+64) |
| 163   | R2, properly paired, mate on reverse strand (1+2+32+128) |
| 77    | R1, unmapped, mate unmapped, not properly paired (1+4+8+64) |
| 141   | R2, unmapped, mate unmapped, not properly paired (1+4+8+128) |
| 4     | Unmapped single-end read |
| 16    | Single-end read on reverse strand |
| 2048  | Supplementary alignment (split read, chimeric) |
| 2064  | Supplementary on reverse strand (2048+16) |

Decode any flag at the command line: `samtools flags 99` (prints flag meaning).

### Common Optional Tags

| Tag | Type | Description |
|-----|------|-------------|
| NM  | i    | Edit distance to the reference (mismatches + indels) |
| MD  | Z    | String encoding mismatched and deleted reference bases |
| AS  | i    | Alignment score (aligner-specific) |
| XS  | i    | Suboptimal alignment score (BWA) / strand for RNA-seq (STAR) |
| SA  | Z    | Supplementary alignments for chimeric reads (semicolon-delimited) |
| HP  | i    | Haplotype tag (used in haplotype-phased BAMs, e.g. HiFi) |
| RG  | Z    | Read group identifier (links to @RG header line) |
| CB  | Z    | Cell barcode (single-cell data) |
| UB  | Z    | UMI barcode |
| tp  | A    | Alignment type: P (primary), S (secondary), I (inversion) — minimap2 |
| de  | f    | Gap-compressed sequence divergence — minimap2 |
| dp  | i    | Total number of alignments — minimap2 long-read |
| rl  | i    | Length of query sequence — minimap2 |
| cg  | Z    | CIGAR string in PAF format — minimap2 |
| cs  | Z    | Difference string — minimap2 |

### CIGAR Operations

| Op | Char | Consumes Query | Consumes Reference | Description |
|----|------|----------------|--------------------|-------------|
| 0  | M    | Yes            | Yes                | Alignment match (can be seq match or mismatch) |
| 1  | I    | Yes            | No                 | Insertion to the reference |
| 2  | D    | No             | Yes                | Deletion from the reference |
| 3  | N    | No             | Yes                | Skipped region (intron in RNA-seq) |
| 4  | S    | Yes            | No                 | Soft clip (bases in SEQ, not aligned) |
| 5  | H    | No             | No                 | Hard clip (bases not in SEQ) |
| 6  | P    | No             | No                 | Padding (silent deletion from padded reference) |
| 7  | =    | Yes            | Yes                | Sequence match (exact) |
| 8  | X    | Yes            | Yes                | Sequence mismatch |

**Rules:**
- CIGAR length = sum of operations that consume the query
- Reference span = sum of operations that consume the reference
- `H` can only appear at the start or end of a CIGAR
- `S` must be adjacent to `H` if both present; otherwise outermost

**Example:** `5S50M3I42M2D5M3S` means:
- 5 bp soft-clipped at start
- 50 bp aligned
- 3 bp inserted into read (present in read, absent in reference)
- 42 bp aligned
- 2 bp deleted from read (gap in reference)
- 5 bp aligned
- 3 bp soft-clipped at end
- Query length = 5+50+3+42+5+3 = 108 bp
- Reference span = 50+42+2+5 = 99 bp

---

## AGP Format Specification

AGP (A Golden Path) is a tab-delimited format describing how an assembly is
constructed from component sequences (contigs/scaffolds) and gaps.

**Version:** AGP 2.1 is current (NCBI standard). AGP 2.0 is common in older VGP
assemblies.

### Column Definitions — Sequence Lines (component type W)

| Col | Name              | Description |
|-----|-------------------|-------------|
| 1   | object            | Name of the object being assembled (chromosome or scaffold) |
| 2   | object_beg        | Start of the component in the object (1-based) |
| 3   | object_end        | End of the component in the object (1-based, inclusive) |
| 4   | part_number       | Line counter within the object (sequential integers, 1-based) |
| 5   | component_type    | W = WGS contig; A = active finishing; D = draft; F = finished; N = gap (known size); U = gap (unknown size) |
| 6   | component_id      | Accession or ID of the component sequence |
| 7   | component_beg     | Start within the component (1-based) |
| 8   | component_end     | End within the component (1-based, inclusive) |
| 9   | orientation       | +, -, ?, na (unknown, unoriented) |

### Column Definitions — Gap Lines (component types N and U)

| Col | Name           | Description |
|-----|----------------|-------------|
| 1   | object         | Name of the object |
| 2   | object_beg     | Start of gap in the object (1-based) |
| 3   | object_end     | End of gap in the object (1-based, inclusive) |
| 4   | part_number    | Sequential line counter |
| 5   | component_type | N = gap with specified size; U = gap of unknown size (default 100 bp) |
| 6   | gap_length     | Length of the gap in bp. 100 for U gaps by convention. |
| 7   | gap_type       | See gap type table below |
| 8   | linkage        | yes / no — is there evidence the flanking sequences are linked? |
| 9   | linkage_evidence | Comma-separated list of evidence types (see below) |

### Gap Types

| Gap Type        | Meaning |
|-----------------|---------|
| scaffold        | Gap between contigs in a scaffold (most common) |
| contig          | Gap between two contigs where linkage is unknown |
| centromere      | Gap for centromere (not fully assembled) |
| short_arm       | Gap for short arm of acrocentric chromosome |
| heterochromatin | Gap for heterochromatin region |
| telomere        | Gap for telomere |
| repeat          | Unresolvable repeat region |
| contamination   | Contaminating sequence removed |
| unknown         | Gap type not specified |

### Linkage Evidence Types

| Evidence            | Meaning |
|---------------------|---------|
| na                  | No linkage evidence (use with linkage=no) |
| paired-ends         | Paired-end read coverage spans the gap |
| align_genus         | Alignment to same genus supports gap size |
| align_xgenus        | Alignment to other genus supports gap size |
| align_trnscpt       | Transcript alignment spans the gap |
| within_clone        | Sequences within same clone |
| clone_contig        | Clone contig map supports the gap |
| map                 | Genetic or physical map supports placement |
| proximity_ligation  | Hi-C or other proximity ligation (most common for scaffolded assemblies) |
| strobe              | Strobe sequencing supports linkage |
| unspecified         | Evidence exists but type not specified |

### Coordinate Validation Rules

For every non-gap line:
```
object_end - object_beg + 1 == component_end - component_beg + 1
```

For the full object, the last `object_end` must equal the sum of all component
lengths plus all gap lengths.

**Worked example:**

```
chr1  1       1000000  1  W  contig001  1  1000000  +
chr1  1000001 1000200  2  N  200        scaffold  yes  proximity_ligation
chr1  1000201 2500000  3  W  contig002  1  1499800  -
```

- Line 1: `1000000 - 1 + 1 = 1000000` and component length `1000000 - 1 + 1 = 1000000` ✓
- Line 2: gap_length = `1000200 - 1000001 + 1 = 200` ✓
- Line 3: `2500000 - 1000201 + 1 = 1499800` and component length `1499800 - 1 + 1 = 1499800` ✓

### Unloc Scaffold Conventions

Unlocalized (`_unloc`) scaffolds are sequences assigned to a chromosome but with
unknown position:
- Naming: `chr1_<scaffold_id>_unloc` or per-genome convention
- In VGP assemblies: often listed after the main chromosome scaffold in the AGP
- Process: scaffolds that Hi-C places on a chromosome but cannot be ordered/
  oriented with confidence relative to other components
- These must be listed as separate objects in the AGP (not embedded in the
  chromosome object)

---

## FASTQ Encoding

### Phred Quality Score

`Q = -10 × log₁₀(P_error)`

| Q score | Error probability | Accuracy |
|---------|-------------------|----------|
| Q10     | 1 in 10           | 90%      |
| Q20     | 1 in 100          | 99%      |
| Q30     | 1 in 1,000        | 99.9%    |
| Q40     | 1 in 10,000       | 99.99%   |
| Q60     | 1 in 1,000,000    | 99.9999% |

### Phred+33 (Sanger / Illumina 1.8+ — current standard)

ASCII encoding: `char = Q + 33`

| Q  | ASCII | Char |
|----|-------|------|
| 0  | 33    | !    |
| 5  | 38    | &    |
| 10 | 43    | +    |
| 20 | 53    | 5    |
| 30 | 63    | ?    |
| 40 | 73    | I    |
| 41 | 74    | J    |
| 93 | 126   | ~    |

Valid range: `!` (Q0) to `~` (Q93).

### Phred+64 (Legacy Illumina 1.3–1.7 — rarely seen post-2011)

ASCII encoding: `char = Q + 64`

| Q  | ASCII | Char |
|----|-------|------|
| 0  | 64    | @    |
| 10 | 74    | J    |
| 40 | 104   | h    |

If you see quality strings starting with `@`, `A`, `B`, suspect Phred+64.
Convert with: `seqtk seq -Q64 -V input.fq > output.fq`

---

## Tool Command Reference

### `samtools view` — filter, convert, subsample

```bash
samtools view [options] <in.bam> [region...]
```

**Key options:**

| Option       | Meaning |
|--------------|---------|
| `-b`         | Output BAM |
| `-C`         | Output CRAM |
| `-h`         | Include header in output |
| `-H`         | Print header only |
| `-c`         | Print only read count (no reads) |
| `-f INT`     | Require these FLAG bits to be set |
| `-F INT`     | Require these FLAG bits to be unset |
| `-q INT`     | Minimum MAPQ |
| `-l STR`     | Only reads in read group STR |
| `-L FILE`    | Only reads overlapping regions in BED file |
| `-s FLOAT`   | Subsample fraction (0.0–1.0); prepend INT for seed: `42.1` = seed 42, 10% |
| `-@ INT`     | Number of compression/decompression threads |
| `-m INT`     | Minimum query length |
| `-e STR`     | Filter expression (e.g. `[NM] <= 5`) |

**Common patterns:**

```bash
# Keep only properly paired primary alignments
samtools view -b -f 2 -F 256 -F 2048 input.bam > output.bam

# Filter by MAPQ and region
samtools view -b -q 20 -L target.bed input.bam > output.bam

# Count mapped reads
samtools view -c -F 4 input.bam

# Extract a specific chromosome
samtools view -b input.bam chr1 > chr1.bam

# Convert to SAM for inspection
samtools view -h input.bam | head -100

# Subsample 10% with fixed seed
samtools view -b -s 42.1 input.bam > subset.bam
```

### `samtools fastx` — BAM/CRAM to FASTQ/FASTA

```bash
samtools fastq [options] <in.bam>
samtools fasta [options] <in.bam>
```

**Key options:**

| Option          | Meaning |
|-----------------|---------|
| `-1 FILE`       | R1 output file |
| `-2 FILE`       | R2 output file |
| `-0 FILE`       | Unpaired/single-end output |
| `-s FILE`       | Singletons (reads whose mates were filtered) |
| `-n`            | Use numeric suffix (/1, /2) |
| `-O`            | Use quality from OQ tag if available |
| `-t`            | Copy RG, BC, QT tags to FASTQ comment |
| `--i1-flags`    | Require these FLAG bits for R1 inclusion |
| `--i2-flags`    | Require these FLAG bits for R2 inclusion |

**Patterns:**

```bash
# Paired-end extraction (both mates must be proper pairs)
samtools fastq -1 R1.fq.gz -2 R2.fq.gz --i1-flags 2 -@ 4 input.bam

# Name-sorted BAM to paired FASTQ (most reliable)
samtools sort -n input.bam | samtools fastq -1 R1.fq.gz -2 R2.fq.gz -0 /dev/null -s /dev/null

# Single-end to FASTA
samtools fasta input.bam > reads.fa
```

### `bamtools filter` — JSON-based BAM filtering

```bash
bamtools filter [-in <filename>] [-out <filename>] [-script <filename>]
```

**Filter keys:**

| Key                | Type | Example value |
|--------------------|------|---------------|
| isPaired           | bool | true          |
| isProperPair       | bool | true          |
| isMapped           | bool | true          |
| isMateMapped       | bool | true          |
| isPrimaryAlignment | bool | true          |
| isDuplicate        | bool | false         |
| isFailedQC         | bool | false         |
| mapQuality         | int  | ">=20"        |
| insertSize         | int  | "<=1000"      |
| tag                | str  | "NM:<=5"      |

**Example filter script (`filter.json`):**

```json
{
  "filters": [
    {
      "isProperPair": true,
      "mapQuality": ">=20",
      "isPrimaryAlignment": true,
      "isDuplicate": false
    }
  ]
}
```

```bash
bamtools filter -script filter.json -in input.bam -out output.bam
```

**Note:** `bamtools isProperPair` applies the full proper-pair definition from
the BAM spec, which is stricter than `samtools view -f 2`. Prefer it for
Hi-C/paired-end pair validation.

### `minimap2` — long-read and splice-aware aligner

```bash
minimap2 [options] <ref.fa> <reads.fa/fq> > output.paf
minimap2 -a [options] <ref.fa> <reads.fa/fq> | samtools sort > output.bam
```

**Presets (`-x`):**

| Preset     | Use case |
|------------|----------|
| map-hifi   | PacBio HiFi (CCS) reads |
| map-pb     | PacBio CLR (continuous long reads) |
| map-ont    | Oxford Nanopore reads |
| sr         | Short-read paired-end alignment |
| asm5       | Assembly-to-assembly, ≤5% divergence |
| asm10      | Assembly-to-assembly, ≤10% divergence |
| asm20      | Assembly-to-assembly, ≤20% divergence |
| splice     | RNA-seq splice-aware |
| splice:hq  | Long-read RNA-seq (Iso-Seq) |

**Key options:**

| Option            | Meaning |
|-------------------|---------|
| `-a`              | Output SAM (default is PAF) |
| `-t INT`          | Threads |
| `--MD`            | Output MD tag |
| `--cs`            | Output cs (difference string) tag |
| `-Y`              | Soft-clip supplementary alignments |
| `-L`              | Write long CIGARs (>65535 ops) to CG tag |
| `--secondary=no`  | Suppress secondary alignments |

**Standard HiFi alignment:**

```bash
minimap2 -a -x map-hifi --MD -t 16 reference.fa reads.fasta.gz \
  | samtools sort -@ 16 -o aligned.bam
samtools index aligned.bam
```

### `BWA-MEM2` — short-read aligner (Hi-C / Illumina)

```bash
bwa-mem2 index reference.fa
bwa-mem2 mem [options] reference.fa R1.fq.gz R2.fq.gz | samtools sort > output.bam
```

**Key options:**

| Option    | Meaning |
|-----------|---------|
| `-t INT`  | Threads |
| `-5`      | Hi-C: report split hit with smallest coordinate as primary |
| `-SP`     | Hi-C: skip mate rescue and pairing |
| `-R STR`  | Read group header line |
| `-M`      | Mark shorter split hits as secondary (Picard-compatible) |

**Standard Hi-C alignment:**

```bash
bwa-mem2 mem -t 16 -5SP reference.fa R1.fq.gz R2.fq.gz \
  | samtools view -bS -F 2316 \
  | samtools sort -n -@ 16 -o hic_namesort.bam
```

`-F 2316` excludes FLAG bits 4 + 8 + 256 + 2048 = 2316:
- 4 (unmapped): keep only mapped reads
- 8 (mate unmapped): keep only reads whose mate also mapped
- 256 (secondary alignment): keep only primary alignments
- 2048 (supplementary/chimeric): exclude split-read artefacts

**Important:** We do NOT require proper pairs (`-f 2`) for Hi-C. Hi-C mates
intentionally map to distant loci — often different chromosomes — which violates
the "proper pair" definition. Requiring proper pairs here would remove the
long-range contacts that are the entire signal.

The `-5SP` flags are required for Hi-C:
- `-5`: always reports the 5'-most alignment as primary (orientation convention)
- `-SP`: disables paired-end rescue — Hi-C mates are expected to map far apart

---

## Sequencing Technology Specifications

### PacBio HiFi (CCS — Circular Consensus Sequencing)

**Key characteristics:**
- Read length: 10–25 kb typical; can reach 30+ kb
- Accuracy: > 99.9% per base (Q20+); typical mean > Q30
- Chemistry: Single-Molecule Real-Time (SMRT); circular library
- CCS: multiple passes around the same molecule → consensus reduces errors
- Typical pass count for Q20+: ≥ 3 passes; optimum 5–10 passes

**File formats:**
- Raw: `.subreads.bam` — do not use for assembly
- Processed: `.hifi_reads.bam` or `.ccs.bam` — use for assembly
- Tag `np`: number of CCS passes
- Tag `rq`: read quality float (0–1; > 0.99 = Q20)

**QC targets:**
- Read length N50: ≥ 15 kb for de novo assembly
- Mean accuracy: > Q20 (filter with `--min-rq 0.99`)
- Coverage: 30–50× for diploid mammalian assembly
- Quick check: `seqkit stats -N 50 reads.fasta.gz`

**Error profile:**
- Primary error type: homopolymer indels (A/T repeats)
- No systematic strand bias
- MAPQ = 0 for repetitive regions — expected, not a mapping failure

### Hi-C (Chromatin Conformation Capture)

**Key characteristics:**
- Short paired-end reads (100–150 bp each end)
- Pairs map to distant loci by design (3D chromatin contacts)
- Restriction enzyme variants:
  - DpnII/MboI (4-cutter, `GATC`): high contact density
  - HindIII (6-cutter, `AAGCTT`): older VGP libraries
  - Arima kit (dual-enzyme, `GATC` + `GANTC`): current VGP standard

**Expected ligation junctions:**

| Enzyme  | Cut Site | Ligation Junction |
|---------|----------|-------------------|
| DpnII   | GATC     | GATCGATC          |
| MboI    | GATC     | GATCGATC          |
| HindIII | AAGCTT   | AAGCTAGCTT        |
| Arima   | GATC+GANTC | GATCGANTC variants|

**Expected mapping statistics:**
- Overall mapping rate: 85–95%
- Trans-chromosomal pairs: 10–30% (normal; pairs on different chromosomes)
- Dangling ends: ≤ 20%

**Artefact types (filtered by Hi-C pipelines):**
- **Dangling ends:** RE site re-ligation without capture
- **Self-circles:** intramolecular ligation; both reads point inward at short distance
- **Self-ligations:** both reads within 1 kb of the same RE site

### Illumina Short Reads

**Key characteristics:**
- Read length: 50–300 bp per end (most common: 150 bp PE)
- Accuracy: Q30 > 80% typical; Q20 > 99%
- Error profile: highest at 3' end; A/C substitution bias late in read

**Common adapter sequences (trim before alignment):**

| Kit        | R1 3' adapter |
|------------|---------------|
| TruSeq     | `AGATCGGAAGAGCACACGTCTGAAC` |
| Nextera/XT | `CTGTCTCTTATACACATCT` |
| TruSeq R2  | `AGATCGGAAGAGCGTCGTGTAGGGA` |

**Trimming tools:** fastp (recommended), Trimmomatic, cutadapt

**Expected mapping rate:** > 95% to a correct reference. < 80% → contamination,
reference mismatch, or adapter-dominated data.

---

## Assembly Quality Metrics

### N50 / L50 / NG50

**N50:** Shortest contig at which ≥ 50% of the total assembly is in contigs of
that length or longer. Higher = more contiguous.

**L50:** Number of contigs needed to reach N50. Lower = more contiguous.

**NG50:** Like N50 but denominator is the expected genome size, not assembly size.
Preferred for cross-species comparisons. NG50 < N50 → assembly is larger than
expected genome (possible haplotype duplication).

**Calculation example:**
Lengths sorted descending: [1000, 800, 600, 400, 200] → total = 3000
50% of 3000 = 1500. Cumulative: 1000, then 1800 ≥ 1500. N50 = 800, L50 = 2.

### BUSCO

BUSCO assesses completeness by checking conserved single-copy orthologs.

**Lineage selection:**

| Organism        | Recommended lineage |
|-----------------|---------------------|
| Mammals         | mammalia_odb10 |
| Birds           | aves_odb10 |
| Fish            | actinopterygii_odb10 |
| Insects         | insecta_odb10 |
| Flowering plants| embryophyta_odb10 |
| Fungi           | fungi_odb10 |
| Bacteria        | bacteria_odb10 |

Use the most specific lineage available; more specific = more relevant gene set.

**Output categories:** C (Complete; S = single, D = duplicated), F (Fragmented),
M (Missing)

**Completeness thresholds:**

| Status           | Target |
|------------------|--------|
| Chromosome-level | C ≥ 95% |
| Scaffold-level   | C ≥ 90% |
| Contig-level     | C ≥ 85% |

High duplication (D > 5%) suggests collapsed haplotypes or genuinely duplicated
lineage (polyploids, ancient whole-genome duplication).

### QV (Quality Value) from Merqury

`QV = -10 × log₁₀(error_rate)` where `error_rate = 1 − (k-mer completeness / 100)`

| QV   | Error rate     |
|------|----------------|
| QV40 | 1 in 10,000    |
| QV50 | 1 in 100,000   |
| QV60 | 1 in 1,000,000 |

VGP target: QV ≥ 40. VGP v2+ chromosome-level: QV ≥ 50.

**Key Merqury outputs:**
- `*.qv` — per-sequence QV
- `*.completeness.stats` — k-mer completeness per haplotype
- `*.spectra-cn.fl.png` — copy-number k-mer spectra

### GenomeScope2

Models k-mer frequency histogram to estimate genome size, heterozygosity, ploidy.

| Output                  | Meaning |
|-------------------------|---------|
| Genome Haploid Length   | Estimated haploid genome size |
| Heterozygosity          | % polymorphic sites between haplotypes |
| Model Fit (R²)          | > 0.9 = good fit |

Heterozygosity > 2%: highly heterozygous; assembly will inflate at haploid
representation unless phased (Hifiasm, HiCanu).

---

## Coverage Calculations

**Formula:** `Coverage (×) = total_bases_sequenced / genome_size`

Equivalently: `Coverage = (read_count × mean_read_length) / genome_size`

**Worked examples:**

*HiFi for 2.5 Gb mammalian genome, 40× target:*
- Required data: `40 × 2.5 Gb = 100 Gb`
- At 15 kb mean read length: `100 Gb / 15 kb ≈ 6.7M reads`
- SMRT Cell 8M yields ~50–70 Gb HiFi → ~2 cells

*Hi-C scaffolding, 50× target on 2.5 Gb:*
- Required data: `50 × 2.5 Gb = 125 Gb`
- At 150 bp PE (300 bp/fragment): `125 Gb / 300 bp ≈ 417M read pairs`
- NovaSeq S4 lane: ~400–600M PE reads → ~1 lane

**Interpreting `samtools coverage` output:**

```
samtools coverage input.bam
# rname  startpos  endpos   numreads  covbases  coverage  meandepth  meanbaseq  meanmapq
# chr1   1         248956422  ...      ...       97.3      43.2       38         60
```

- `coverage`: % of bases covered by ≥ 1 read
- `meandepth`: mean read depth across covered region
- Low `meanmapq` (< 30) on a chromosome: repeat-heavy region or wrong reference

---

## Coordinate Systems — Extended

| System            | First base | Interval type    | Example (bases 2,3,4,5) | Tools |
|-------------------|------------|------------------|--------------------------|-------|
| 1-based closed    | 1          | [start, end]     | start=2, end=5           | SAM, VCF, GFF, AGP, IGV, UCSC, Ensembl |
| 0-based half-open | 0          | [start, end)     | start=1, end=5           | BED, BAM binary, bedtools, Python |

**Conversion:**

```
SAM/VCF → BED:   BED_start = SAM_start - 1;  BED_end = SAM_end
BED → SAM/VCF:   SAM_start = BED_start + 1;  SAM_end = BED_end
```

**Common off-by-one traps:**

1. `samtools view chr1:100-200` uses 1-based closed. Equivalent BED: `chr1  99  200`.
2. Converting BED intersections back to VCF: add 1 to bedtools start coordinate.
3. AGP to BED: subtract 1 from `object_beg` column when writing BED.
4. Python indexing: SAM POS 100 → `seq[99]`; a 100 bp span ending at POS 200 → `seq[99:200]`.

**Tool-specific coordinate conventions:**

| Tool       | Coordinates       | Notes |
|------------|-------------------|-------|
| samtools   | 1-based closed    | Region syntax: `chr1:100-200` |
| bedtools   | 0-based half-open | BED throughout |
| IGV        | 1-based closed    | GUI display; BED files auto-converted |
| UCSC browser | 1-based closed  | Table browser exports; BED is 0-based |
| Ensembl    | 1-based closed    | REST API |
| VCF POS    | 1-based           | Indel REF/ALT anchored at POS |
| GFF/GTF    | 1-based closed    | Start and end inclusive |
| BAM binary | 0-based           | Internal; samtools view displays as 1-based |

---

## Oxford Nanopore Sequencing (ONT)

### Key Characteristics
- Read length: R9.4.1 and R10.4.1 pores; typically 10–100 kb; ultra-long reads up to 4 Mb
- Accuracy: R9.4.1 with Guppy high-accuracy = ~98–99%; R10.4.1 = ~99.5%; duplex mode = ~99.9%
- Error profile: context-dependent errors (homopolymers, low-complexity), systematic errors in certain motifs (e.g. 5mCpG methylation affects raw signal and can mimic errors)
- Not paired-end: like HiFi, Nanopore is single-end (each pore reads one strand or duplex)

### Basecalling: Dorado (current standard)
Dorado replaced Guppy as the primary ONT basecaller in 2023.

```bash
# Standard basecalling (high-accuracy model)
dorado basecaller hac pod5_directory/ > reads.bam

# Super-accuracy model (slower, highest accuracy)
dorado basecaller sup pod5_directory/ > reads.bam

# Simplex with 5mCpG methylation calling
dorado basecaller hac,5mCG_5hmCG pod5_directory/ > reads_methyl.bam

# Duplex calling (two complementary strands together; highest accuracy)
dorado duplex hac pod5_directory/ > duplex.bam
```

Model selection:
- `fast`: lowest accuracy, fastest (not recommended for most analyses)
- `hac` (high accuracy): standard choice, good speed/accuracy balance
- `sup` (super accuracy): best accuracy, 3-5× slower; use for final publication-quality calls
- Models are chemistry-specific: R9.4.1 and R10.4.1 have separate model sets; always match model to flow cell chemistry

### QC Metrics for ONT Data
```bash
# NanoStat summary
NanoStat --fastq reads.fastq.gz --outdir nanostat_out/

# NanoPlot for quality plots
NanoPlot --fastq reads.fastq.gz --outdir nanoplot_out/ --plots dot

# seqkit stats
seqkit stats -N 50 reads.fastq.gz
```

Key QC metrics:
| Metric | Target | Red flag |
|--------|--------|----------|
| Mean read quality | > Q15 (hac) / > Q20 (sup) | < Q10 (degrade with age) |
| Read N50 | > 20 kb (genomic) | < 5 kb (fragmentation) |
| % reads > Q10 | > 80% | < 60% |
| Mean read length | 20–50 kb (depends on library) | < 5 kb (over-shearing) |

### Alignment
```bash
# Standard genomic alignment
minimap2 -a -x map-ont --MD -t 16 reference.fa reads.fastq.gz \
  | samtools sort -@ 16 -o aligned.bam
samtools index aligned.bam

# Ultra-long read alignment (>100 kb reads)
minimap2 -a -x map-ont -L --MD -t 16 reference.fa reads.fastq.gz \
  | samtools sort -@ 16 -o aligned.bam
# -L: write CIGAR with >65535 ops to CG tag (required for ultra-long reads)
```

### Error Profile and Downstream Impact
- Homopolymer errors: runs of 4+ identical bases (AAAA, TTTT) frequently have 1-base indels. Avoid using raw ONT data for precise indel calling in homopolymers.
- Systematic errors in specific 6-mer contexts (see ONT technical data; varies by model)
- For variant calling: use Clair3 (designed for ONT; handles systematic errors) or DeepVariant (ONT model available)
- For SV calling: Sniffles2 (long-read SV caller) or SVABA

---

## Structural Variant (SV) Calling

### Types of Structural Variants

| SV Type | Size | Description | Detection signal |
|---------|------|-------------|-----------------|
| Deletion | >50 bp | Sequence present in reference, absent in sample | Split reads, discordant pairs, read depth drop |
| Insertion | >50 bp | Sequence absent in reference, present in sample | Split reads, discordant pair distance increase |
| Inversion | >50 bp | Segment flipped in orientation | Discordant read-pair orientation |
| Duplication | >50 bp | Tandem or interspersed copy number gain | Read depth increase, discordant pairs |
| Translocation | Any | Sequence moved between chromosomes | Inter-chromosomal discordant pairs |
| Mobile element insertion | 100–6000 bp | Transposable element insertion | Split reads with TE sequence |

### Short-Read SV Callers

**Manta** (Illumina, recommended for germline and somatic):
```bash
# Configure
manta/bin/configManta.py --bam sample.bam --referenceFasta ref.fa --runDir manta_out/
# Run
manta_out/runWorkflow.py -m local -j 16
# Output: manta_out/results/variants/diploidSV.vcf.gz (germline)
#                                    somaticSV.vcf.gz (somatic, if tumor+normal)
```
Manta detects: deletions, insertions, tandem dups, inversions, translocations. Strong for insertions from split-read evidence.

**DELLY** (general-purpose, good for rare SVs):
```bash
delly call -g ref.fa -o svs.bcf sample.bam
delly filter -f germline -o filtered.bcf svs.bcf
bcftools convert -O v filtered.bcf > filtered.vcf
```

**SMOOVE** (Lumpy wrapper, good for cohort SV calling):
```bash
smoove call --outdir smoove_out/ --name sample --fasta ref.fa --procs 8 sample.bam
```

### Long-Read SV Callers

**Sniffles2** (HiFi and ONT):
```bash
# Single sample
sniffles --input aligned.bam --vcf svs.vcf --reference ref.fa

# Multi-sample (population mode — more accurate)
# Step 1: per-sample SNFL files
sniffles --input sample1.bam --snf sample1.snf --reference ref.fa
sniffles --input sample2.bam --snf sample2.snf --reference ref.fa
# Step 2: joint calling
sniffles --input sample1.snf sample2.snf --vcf cohort_svs.vcf
```

Long-read SV calling advantages:
- Insertions fully resolved (short reads can only detect insertion size, not sequence)
- Complex SVs (chromoplexy, chromothripsis) visible as multi-breakpoint events
- Mobile element insertions identified by sequence

### SV Genotyping
Called SVs need genotyping in all samples for population analysis:
```bash
# Genotype Manta SVs in additional samples
manta/bin/configManta.py --bam additional_sample.bam --referenceFasta ref.fa \
  --runDir geno_out/ --forceSV manta_svs.vcf.gz
```

### SV Validation and Filtering
- Minimum support: ≥3–5 reads supporting the SV for high-confidence calls
- Population frequency: rare SVs (AF < 1%) more likely to be functional
- SURVIVOR: merge SV calls from multiple callers; SVs supported by ≥2 callers are more reliable
```bash
ls *.vcf > vcf_list.txt
SURVIVOR merge vcf_list.txt 1000 2 1 0 0 50 merged_svs.vcf
# 1000: max distance between breakpoints; 2: min callers; 50: min SV size
```

---

## Variant Annotation

After calling variants (SNPs, indels, SVs), annotation links genomic positions to functional impact and known databases.

### VEP (Ensembl Variant Effect Predictor)

VEP is the most comprehensive variant annotation tool. It annotates:
- Variant consequence (missense, synonymous, splice-site, intergenic, etc.)
- Gene and transcript impact (HGVS notation)
- Population frequencies (gnomAD, ExAC, 1000 Genomes)
- Pathogenicity scores (CADD, SIFT, PolyPhen-2)
- Clinical significance (ClinVar)

```bash
# Install with cache (recommended for speed)
vep --cache --dir_cache ~/.vep \
  --input_file variants.vcf \
  --output_file annotated.vcf \
  --vcf \
  --species homo_sapiens \
  --assembly GRCh38 \
  --everything           # enable all annotations
  --fork 8               # parallel threads

# Key flags
# --pick: output one consequence per variant (most severe)
# --canonical: flag canonical transcript
# --af_gnomad: add gnomAD population frequencies
# --sift b: SIFT prediction + score
# --polyphen b: PolyPhen-2 prediction + score
# --cadd_snvs: CADD scores (requires local installation)
```

**VEP consequence terms (most to least severe):**
transcript_ablation > splice_acceptor_variant > splice_donor_variant > stop_gained > frameshift_variant > stop_lost > start_lost > transcript_amplification > inframe_insertion > inframe_deletion > missense_variant > protein_altering_variant > splice_region_variant > incomplete_terminal_codon_variant > start_retained_variant > stop_retained_variant > synonymous_variant > coding_sequence_variant > mature_miRNA_variant > 5_prime_UTR_variant > 3_prime_UTR_variant > non_coding_transcript_exon_variant > intron_variant > NMD_transcript_variant > non_coding_transcript_variant > upstream_gene_variant > downstream_gene_variant > TFBS_ablation > TFBS_amplification > TF_binding_site_variant > regulatory_region_ablation > regulatory_region_amplification > feature_elongation > regulatory_region_variant > feature_truncation > intergenic_variant

### SnpEff

Simpler than VEP; uses its own database; better for non-human organisms.

```bash
# List available databases
snpeff databases | grep -i "homo_sapiens"

# Annotate
snpeff GRCh38.99 variants.vcf > annotated.vcf 2> snpeff.log

# For non-model organisms: build database from GTF + FASTA
snpeff build -gtf22 -v mySpecies
```

### Population Frequency Filtering

A variant's population frequency is critical for disease interpretation:

| gnomAD AF | Interpretation for rare Mendelian disease |
|-----------|------------------------------------------|
| > 1% | Very unlikely disease-causing (common variant) |
| 0.1–1% | Low frequency; possible risk allele; unlikely for rare recessive |
| < 0.1% | Rare; warrants investigation |
| < 0.01% | Very rare; strong candidate for rare Mendelian disease |
| Not in gnomAD | Novel; high priority for rare disease (but may be sequencing artefact) |

```bash
# Filter to rare variants (<1% in any gnomAD population)
bcftools filter -i 'gnomAD_AF < 0.01 || gnomAD_AF = "."' annotated.vcf > rare.vcf
```

### Pathogenicity Scores

| Score | Range | Interpretation |
|-------|-------|----------------|
| CADD Phred | 0–99 | > 20 = top 1% most deleterious; > 30 = top 0.1% |
| SIFT | 0–1 | < 0.05 = damaging; > 0.05 = tolerated |
| PolyPhen-2 | 0–1 | > 0.908 = probably damaging; 0.446–0.908 = possibly damaging |
| GERP++ RS | -12.3 to 6.17 | > 2 = conserved; > 4 = highly conserved |
| PhyloP | -14 to 6 | > 2 = conserved across vertebrates |

No single score is definitive. Use in combination. CADD is the most broadly applicable.

---

## Hi-C Scaffolding Tools

Hi-C contact data is used to order and orient contigs/scaffolds into chromosome-scale assemblies. Three major tools, each with different algorithms and trade-offs.

### YAHS (Yet Another Hi-C Scaffolding Tool)

Currently recommended for VGP-style assemblies (2022–present). Fastest, most robust to noisy Hi-C data.

```bash
yahs reference.fa hic_reads.bam -o scaffolded
# Input: contig-level assembly FASTA + Hi-C BAM (name-sorted)
# Output: scaffolded.agp, scaffolded_scaffolds_final.fa
```

Key advantages:
- Handles mixed-ploidy assemblies (haplotype-resolved with Hifiasm + Hi-C)
- Produces AGP output directly
- More tolerant of low coverage Hi-C
- Fastest of the three tools

### SALSA2 (Scaffolding using Alignment and Length Scaled Analysis)

Graph-based algorithm; good for complex genomes with many small contigs.

```bash
# Requires: BED file of Hi-C alignments (not BAM)
python run_pipeline.py \
  -a contigs.fa \
  -l contigs.fa.fai \
  -b hic.bed \
  -e GATC \
  -o salsa_output/
```

Key advantages:
- Better for highly fragmented assemblies (thousands of contigs)
- Graph-based: explicitly models uncertainty in contig ordering
- Produces confidence scores for scaffold joins

### 3D-DNA (Three-Dimensional DNA Organization)

Used in Juicer pipeline; historically important (many published VGP v1 assemblies used it).

```bash
# Run after Juicer Hi-C processing
3d-dna/run-asm-pipeline.sh reference.fasta merged_nodups.txt
```

Key advantages:
- Integrated with Juicer (if already using Juicer Hi-C pipeline)
- Good for chromosome-scale misassembly detection
- Produces `.hic` files for visualization in Juicebox

### Comparison

| Tool | Speed | Fragmented assemblies | Misassembly detection | Output |
|------|-------|----------------------|----------------------|--------|
| YAHS | Fastest | Good | Moderate | AGP + FASTA |
| SALSA2 | Moderate | Excellent | Good | FASTA + scaffold info |
| 3D-DNA | Slowest | Moderate | Excellent (Juicebox review) | FASTA + .hic |

**Recommendation:** Start with YAHS for speed and AGP output. Use 3D-DNA + Juicebox review for assemblies where manual curation of misassemblies is needed.

### Post-Scaffolding Manual Curation

Automated Hi-C scaffolding is never perfect. Inspect the contact map:
```bash
# Generate .hic file from YAHS output for Juicebox
juicer_tools pre -s yahs.liftover.agp.txt scaffolded.agp yahs.alignment.bed \
  genome.chrom.sizes scaffolded.hic
# Open scaffolded.hic in Juicebox GUI to review scaffold joins
```

Signs of misassembly in the contact map:
- Cross-shaped artefact at a scaffold join: two unrelated contigs were incorrectly joined
- Missing diagonal: a contig is oriented backwards
- Off-diagonal signal: two distant scaffolds should be adjacent

---

## DNA Methylation (WGBS / ONT)

### Bisulfite Sequencing (WGBS and RRBS)

Whole-genome bisulfite sequencing (WGBS) converts unmethylated cytosines to uracil (read as thymine), while methylated cytosines remain unchanged. This allows direct measurement of CpG methylation at single-base resolution.

**Library types:**
- WGBS: whole-genome coverage, 30–50× typical; most comprehensive
- RRBS (Reduced Representation BS): uses restriction enzyme to enrich CpG-rich regions; cheaper; biased to promoters/CpG islands
- EM-seq (enzymatic methyl-seq): replaces bisulfite chemistry with enzymes; reduces damage to DNA; compatible with standard sequencing depth

### Alignment with Bismark

```bash
# Build bisulfite-converted reference
bismark_genome_preparation reference/

# Align paired-end WGBS
bismark --genome reference/ -1 R1.fq.gz -2 R2.fq.gz --bam --parallel 8

# Extract methylation information
bismark_methylation_extractor --paired-end --CX_context \
  --bedGraph --comprehensive \
  --genome_folder reference/ \
  sample_bismark_bt2_pe.bam

# Output: CpG_context_*.txt (per-CpG methylation + coverage)
#         bismark.bedGraph.gz (methylation % per CpG)
```

### Alignment with Bismark: Key Flags

| Flag | Meaning |
|------|---------|
| `--non_directional` | Library is non-directional (random strand capture); try if mapping rate is low |
| `--pbat` | Post-bisulfite adaptor tagging libraries (PBAT, scBS-seq) |
| `--CX` | Extract all CpG, CHG, CHH contexts (not just CpG) |
| `--ignore` N | Ignore first N bases of each read (trim ends with systematic bias) |

### ONT Methylation Calling (No Bisulfite Required)

Modern ONT basecallers detect methylation directly from the raw signal:

```bash
# Basecall with methylation model
dorado basecaller hac,5mCG_5hmCG pod5_files/ > reads_methyl.bam

# Modkit: extract per-site methylation from modBAM
modkit pileup reads_methyl.bam output_methylation.bed \
  --ref reference.fa \
  --preset traditional
# Output: bedMethyl format (chrom, start, end, context, coverage, % methylated)
```

### Differential Methylation Analysis

**MethylKit** (R, site-level and region-level):
```R
library(methylKit)
# Load methylation data
myobj <- methRead(list("treated.CpG.txt", "control.CpG.txt"),
                  sample.id=list("treated","control"),
                  assembly="hg38", treatment=c(1,0))
# Filter low-coverage sites
filtered <- filterByCoverage(myobj, lo.count=10, lo.perc=NULL, hi.perc=99.9)
# Normalise
normalised <- normalizeCoverage(filtered)
# Find DMRs (differentially methylated regions)
meth <- unite(normalised)
diff <- calculateDiffMeth(meth)
```

**BSmooth** (R/Bioconductor, smooth region-level DMRs):
- Better statistical model for low-coverage WGBS
- Uses local smoothing to borrow strength across neighbouring CpGs
- More robust for RRBS data

### Interpreting Methylation Data

**CpG context interpretation:**
- CpG islands (CGIs): CpG-dense regions in promoters; typically unmethylated in expressed genes; methylation = gene silencing
- Gene body methylation: moderately methylated; positive correlation with transcription (counterintuitive but robust)
- Imprinted regions: one allele methylated, one not (~50% methylation); allele-specific methylation

**Coverage requirements:**
- Minimum 10× per CpG per sample for reliable methylation estimates
- 30× recommended for WGBS; 50× for differentially methylated region (DMR) calling

**Conversion efficiency (critical QC):**
WGBS requires > 99% bisulfite conversion efficiency. Incomplete conversion causes false unmethylated-to-methylated errors.
```bash
# Check conversion efficiency from Bismark report
grep "C methylated in CpHpG context" sample_bismark_bt2_PE_report.txt
# Should be < 1% (unmethylated CHG should all convert)
```
