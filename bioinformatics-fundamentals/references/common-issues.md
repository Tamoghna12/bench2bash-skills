# Bioinformatics Fundamentals — Common Issues & Troubleshooting

Diagnostic guide for the most common silent failures in sequencing data
processing. Each section covers symptom, root cause, fix, and diagnostic commands.

---

## Table of Contents

1. [Empty Output After Region Filtering](#empty-output-after-region-filtering)
2. [Paired-End Read Count Mismatches](#paired-end-read-count-mismatches)
3. [Low Mapping Rate](#low-mapping-rate)
4. [Proper Pairs Lost After Mapping](#proper-pairs-lost-after-mapping)
5. [Hi-C Specific Issues](#hi-c-specific-issues)
6. [HiFi Specific Issues](#hifi-specific-issues)
7. [AGP Processing Errors](#agp-processing-errors)
8. [Format Conversion Issues](#format-conversion-issues)
9. [Diagnostic Commands Reference](#diagnostic-commands-reference)

---

## Empty Output After Region Filtering

**Symptom:** `samtools view -L regions.bed | bamtools filter -isProperPair true`
produces zero reads (or drastically fewer than expected).

**Root cause:** The region filter (`-L`) checks each read in isolation. When
mate pairs map to different chromosomes or scaffolds — common in Hi-C and
expected in paired-end whole-genome data — filtering by region removes one mate.
With its mate gone, the surviving read can no longer form a proper pair, so the
downstream pair filter removes it too. The result is empty output even when
there are thousands of reads in the region.

**Fix:** Always apply the proper-pair filter *before* the region filter:

```bash
# WRONG — breaks pairs first
samtools view -b -L regions.bed input.bam \
  | bamtools filter -isProperPair true > output.bam
# Result: empty or near-empty

# CORRECT — pair filter first, then region filter
samtools view -b -f 2 -L regions.bed input.bam > output.bam
```

**Distinguishing "empty due to pair-breaking" vs "genuinely no reads":**

```bash
# Step 1 — check if region has any reads at all (ignore pairing)
samtools view -c input.bam chr1:1000000-2000000
# If 0: region is genuinely empty; look upstream

# Step 2 — check how many are properly paired before region filter
samtools view -c -f 2 input.bam chr1:1000000-2000000
# If > 0 but filtered output is empty: pair-breaking is the cause

# Step 3 — check mate chromosome for reads in the region
samtools view input.bam chr1:1000000-2000000 | awk '{print $7}' | sort | uniq -c | sort -rn
# High counts of '=' (same chrom) = normal paired-end DNA-seq
# High counts of other chroms = Hi-C trans-reads (expected)
```

**Other tool combinations that also break pairs:**

```bash
# Also broken
bamtools filter -isProperPair true -in input.bam \
  | samtools view -b -L regions.bed > output.bam

# Safe — combined flags in samtools
samtools view -b -f 2 -q 20 -L regions.bed input.bam > output.bam
```

---

## Paired-End Read Count Mismatches

**Symptom:** After `samtools fastq`, `wc -l R1.fastq` ≠ `wc -l R2.fastq`, or
downstream tools fail with "unequal read pair count".

**Root cause:** A filtering step removed one mate of a pair without removing
the other. R1 and R2 files fall out of sync.

**Fix — prevention (best):** Require proper pairs during FASTQ extraction:

```bash
# --i1-flags 2 ensures only reads whose mates are present are output
samtools fastq -1 R1.fq.gz -2 R2.fq.gz --i1-flags 2 input.bam

# Or sort by name first (most reliable for any BAM)
samtools sort -n input.bam \
  | samtools fastq -1 R1.fq.gz -2 R2.fq.gz -0 /dev/null -s /dev/null
```

**Fix — repair already-mismatched files:**

```bash
# BBTools repair.sh (fast, recommended)
repair.sh in=R1.fq.gz in2=R2.fq.gz out=R1_fixed.fq.gz out2=R2_fixed.fq.gz outs=singletons.fq.gz

# pairfq (Perl, no extra dependencies)
pairfq makepairs -f R1.fq -r R2.fq -fp R1_paired.fq -rp R2_paired.fq -fs R1_s.fq -rs R2_s.fq

# fastp (also re-syncs in paired-end mode)
fastp -i R1.fq.gz -I R2.fq.gz -o R1_out.fq.gz -O R2_out.fq.gz --unpaired1 singletons.fq.gz
```

**Verify sync before proceeding:**

```bash
r1_count=$(zcat R1.fq.gz | wc -l)
r2_count=$(zcat R2.fq.gz | wc -l)
echo "R1: $((r1_count/4)) reads, R2: $((r2_count/4)) reads"
[ "$r1_count" -eq "$r2_count" ] && echo "PASS" || echo "FAIL — mismatched"
```

---

## Low Mapping Rate

**Symptom:** `samtools flagstat` shows < 80% mapped reads for genomic DNA-seq,
or tools that expect high mapping rates report failures.

**Decision tree:**

```
Low mapping rate
├── Is this Hi-C data?
│   └── 70–90% mapped is normal (chimeric reads, ligation artefacts)
│       Use Hi-C-specific pipeline (HiC-Pro, Juicer, SALSA2)
│
├── Check for adapter contamination
│   └── Run: fastqc R1.fq.gz R2.fq.gz
│       If "Adapter Content" panel shows adapters:
│       trim with: fastp --detect_adapter_for_pe -i R1.fq.gz -I R2.fq.gz ...
│
├── Is the reference the right species?
│   └── Sample 10 unmapped reads and BLAST against NCBI nt
│       samtools view -f 4 input.bam | head -50 | awk '{print ">"$1"\n"$10}' > unmapped.fa
│       If hits = different species: contamination or wrong reference
│
├── Is the mapper preset correct?
│   ├── HiFi reads → map-hifi (not map-ont, not sr)
│   ├── Illumina PE → bwa-mem2 or minimap2 -x sr
│   └── Wrong preset silently misaligns or drops reads
│
└── Check chromosome naming
    └── samtools view -H input.bam | grep "^@SQ" | head -5
        Ref FASTA headers should match exactly (chr1 vs 1 vs Chr1)
```

**Diagnostic commands:**

```bash
# Overall mapping stats
samtools flagstat input.bam

# Sample unmapped reads for BLAST identification
samtools view -f 4 input.bam | head -100 \
  | awk '{print ">"$1"\n"$10}' > unmapped_sample.fa

# Run FastQC
fastqc --threads 4 --outdir qc/ R1.fq.gz R2.fq.gz

# Compare reference and BAM chromosome names
grep ">" reference.fa | head -5
samtools view -H input.bam | grep "^@SQ" | awk '{print $2}' | sed 's/SN://' | head -5
```

---

## Proper Pairs Lost After Mapping

**Symptom:** `samtools flagstat` shows low % properly paired (< 80% for standard
paired-end genomic data) after alignment.

**Diagnosis workflow:**

```bash
# Step 1 — Full stats including insert size
samtools stats input.bam | grep "^SN" | head -30
# Look for: "reads properly paired", "insert size average", "insert size std dev"

# Step 2 — Check insert size distribution
samtools stats input.bam | grep "^IS" | awk '{print $2, $3}' | head -20
# All zeros or absent = serious mapping problem

# Step 3 — Check flagstat breakdown
samtools flagstat input.bam
# Ratio of "properly paired" to "paired in sequencing"

# Step 4 — Check if R1/R2 are labelled correctly
samtools view -f 67 -c input.bam   # R1, paired, properly paired
samtools view -f 131 -c input.bam  # R2, paired, properly paired
# If one is near-zero: reads are mislabelled or files are swapped

# Step 5 — Check for chromosome naming inconsistency
samtools view input.bam | awk 'NR<=200{print $3, $7}' | awk '$2=="*"' | wc -l
# High count of RNEXT="*": reference naming mismatch, mate cannot be found
```

**Common causes and fixes:**

| Cause | Symptom | Fix |
|-------|---------|-----|
| Reference naming mismatch | RNEXT = `*` frequently | Use same reference everywhere; check chr prefix |
| R1/R2 files swapped | Low proper-pair %, odd insert sizes | Re-run with files in correct order |
| Hi-C inter-chromosomal pairs | Appears "improperly paired" | Normal; use Hi-C-aware filtering |
| Wrong mapper flags for Hi-C | All reads improperly paired | Use `-5SP` with BWA-MEM2 |
| Insert size > contig length | Mate maps to different contig | Expected for highly fragmented assemblies |

---

## Hi-C Specific Issues

### Low Mapping Rate (< 70%)

Chimeric reads (reads spanning a ligation junction) often fail end-to-end
mapping. Use a Hi-C-aware mapper:
- **BWA-MEM2** with `-5SP` (splits chimeric reads at ligation junction)
- **HiC-Pro**: maps in two steps — full read first, then unaligned 5' fragment

### High Dangling-End Rate (> 30%)

Dangling ends = both reads from the same restriction fragment (no ligation
captured). Causes:
- Incomplete restriction digestion → increase enzyme concentration or time
- Over-sonication → reduce fragmentation intensity
- Check HiC-Pro or Juicer QC output:

```bash
# HiC-Pro: check mapping_stats/ directory
cat hic_results/stats/*_mapping_stats.txt | grep -E "Dangling|Self_ligation|Valid"

# Juicer: check inter.txt
grep -E "dangling|self" inter.txt
```

### Low Cis-to-Trans Ratio

Expected: 70–90% of pairs map within the same chromosome. If trans > 40%:
- Degraded chromatin (DNA sheared before fixation)
- Non-chromatin DNA contamination
- Over-digestion

```bash
# Measure cis vs trans ratio
samtools view -f 1 input.bam \
  | awk '{print ($3 == $7 || $7 == "=") ? "cis" : "trans"}' \
  | sort | uniq -c
```

### Name-Sort Required for Most Hi-C Tools

Juicer, SALSA2, and 3D-DNA all require name-sorted BAM so paired reads are
adjacent:

```bash
samtools sort -n -@ 16 mapped.bam -o namesorted.bam
```

---

## HiFi Specific Issues

### CCS vs Subread BAM Confusion

Subread BAMs contain raw circular SMRTbell reads. CCS BAMs contain consensus
reads. **Only use CCS BAMs for assembly.**

```bash
# Identify BAM type by read names
samtools view input.bam | head -3 | awk '{print $1}'
# CCS: m64xxxxx_xxxxxx_xxxxxx/zmw/ccs
# Subreads: m64xxxxx_xxxxxx_xxxxxx/zmw/start_end

# Or check header
samtools view -H input.bam | grep "@RG" | grep -o "DS:[^	]*"
# CCS BAM will contain READTYPE=CCS in the DS tag
```

### MAPQ = 0 for Many HiFi Reads

MAPQ = 0 in minimap2 for reads mapping to repetitive regions. This is expected
and does **not** mean the read is unmapped.

```bash
# How many are MAPQ = 0 but mapped?
samtools view -F 4 input.bam | awk '$5==0{c++} END{print c " MAPQ-0 mapped reads"}'
# If large fraction: repetitive genome (normal for many vertebrates)
# Do NOT filter these out wholesale — you lose real repeat-region signal
```

### Kinetics Tags Inflating File Size

Modern SMRTLink CCS BAMs include per-base kinetics tags (IPD, PW). Strip them
if not needed for methylation analysis:

```bash
samtools view -@ 8 -b --remove-tag fi,ri,fp,rp,ip,pw input.hifi.bam -o reads.nok.bam
samtools index reads.nok.bam
```

### Adapter Contamination in HiFi

SMRTLink ≥ 10 strips adapters automatically. If FastQC shows adapter content:

```bash
# PacBio tool for adapter finding and removal
pbadapterfinder --input reads.bam --output reads.trimmed.bam
```

---

## AGP Processing Errors

### Off-by-One Coordinate Errors

AGP uses 1-based closed intervals. Treating them as 0-based produces off-by-one
errors when extracting sequences:

```bash
# WRONG — position 0 is invalid; samtools uses 1-based coordinates throughout.
# This fetches bases 1–999999 (999999 bp), missing the last base of a 1 Mb contig.
samtools faidx assembly.fa "contig001:0-999999"

# CORRECT — AGP component_beg (col 7) is 1-based; match it exactly.
# This fetches all 1,000,000 bp of the contig.
samtools faidx assembly.fa "contig001:1-1000000"
```

### Component Length Mismatch

Validation rule: `object_end - object_beg + 1 == component_end - component_beg + 1`

```bash
# Quick validation with awk
awk '$5 !~ /^[NU]$/ {
  obj_len = $3 - $2 + 1
  comp_len = $8 - $7 + 1
  if (obj_len != comp_len)
    print "MISMATCH line " NR ": obj=" obj_len " comp=" comp_len
}' assembly.agp
```

### Invalid or Missing Linkage Evidence

For Hi-C-scaffolded gaps, use `proximity_ligation` as linkage evidence:

```
chr1  1000001  1000100  2  N  100  scaffold  yes  proximity_ligation
```

Not providing linkage evidence (or using `na` with `linkage=yes`) will fail
NCBI submission validation.

### Cumulative Coordinate Errors After Editing

When adding or removing components, all subsequent `object_beg`/`object_end`
values must be recalculated. Always use an AGP-aware tool (NCBI's agptools,
or the VGP curation scripts) rather than editing coordinates by hand.

---

## Format Conversion Issues

### BAM to FASTQ: Coordinate-Sorted vs Name-Sorted

Paired-end BAM must be name-sorted before FASTQ conversion for R1/R2 to stay
in sync:

```bash
# WRONG — coordinate-sorted BAM
samtools fastq -1 R1.fq.gz -2 R2.fq.gz coord_sorted.bam
# R1 and R2 will be out of sync

# CORRECT
samtools sort -n -@ 8 coord_sorted.bam \
  | samtools fastq -1 R1.fq.gz -2 R2.fq.gz -0 /dev/null -s /dev/null
```

### FASTQ to FASTA

Stripping quality lines manually breaks when read names contain spaces
(common in Illumina headers). Use dedicated tools:

```bash
# seqtk (fastest)
seqtk seq -a input.fq.gz > output.fa

# samtools
samtools fasta input.bam > output.fa
```

### Coordinate System Conversion (BED to SAM and back)

```bash
# BED region to samtools region syntax (0-based → 1-based)
awk '{print $1 ":" $2+1 "-" $3}' regions.bed

# samtools region to BED (1-based → 0-based half-open)
echo "chr1:100-200" | awk -F'[:-]' '{print $1 "\t" $2-1 "\t" $3}'
```

---

## Diagnostic Commands Reference

### Mapping Quality Assessment

```bash
# Overall mapping statistics
samtools flagstat input.bam

# Detailed statistics including insert size distribution
samtools stats input.bam | grep -E "^SN|^IS" | head -60

# Flag distribution
samtools view -c -f 2 input.bam      # Properly paired
samtools view -c -F 4 input.bam      # Mapped
samtools view -c -f 4 input.bam      # Unmapped
samtools view -c -f 256 input.bam    # Secondary alignments
samtools view -c -f 2048 input.bam   # Supplementary alignments

# Per-chromosome coverage summary
samtools coverage input.bam | column -t | head -30
```

### BAM Structure Checks

```bash
# Check sort order
samtools view -H input.bam | grep "^@HD"

# List reference sequences in BAM header
samtools view -H input.bam | grep "^@SQ" | head -10

# Quick region read count
samtools view -c input.bam chr1:1000000-2000000

# Inspect first few reads with header
samtools view -h input.bam | head -50
```

### Paired-End Validation

```bash
# Check R1/R2 balance
samtools view -c -f 64 input.bam     # R1 count
samtools view -c -f 128 input.bam    # R2 count
# Should be equal for clean paired-end data

# Hi-C cis/trans ratio
samtools view -f 1 input.bam \
  | awk '{print ($3 == $7 || $7 == "=") ? "cis" : "trans"}' \
  | sort | uniq -c
```

### FASTQ Quality Checks

```bash
# Read count and N50 length
seqkit stats -N 50 reads.fq.gz

# Quick quality check (first 10k reads)
seqtk fqchk reads.fq.gz | head -5

# Adapter and quality report
fastqc --threads 4 --outdir qc/ R1.fq.gz R2.fq.gz
```

### Assembly QC

```bash
# Assembly stats
seqkit stats -N 50 -N 90 assembly.fa

# BUSCO (adjust lineage as appropriate)
busco -i assembly.fa -l mammalia_odb10 -o busco_out -m genome --cpu 8

# Merqury QV estimate (requires k-mer database from reads)
meryl k=21 count reads.fasta.gz output reads.meryl
merqury.sh reads.meryl assembly.fa merqury_output

# Per-contig depth (samtools)
samtools coverage -A -w 100 aligned.bam > coverage_windows.txt
```
