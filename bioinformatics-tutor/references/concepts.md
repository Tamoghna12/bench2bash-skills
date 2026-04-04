# Bioinformatics Concepts — The Biological Story Behind Each Domain

For each major domain: the biological question it answers, the measurement, the
computational translation, and how to validate the result biologically.

---

## 1. Genome Assembly

**The biological story:** Every organism's genome is a single (or diploid pair
of) long DNA molecule(s). We can't read it in one go — sequencers produce
millions of short fragments. Assembly is the process of reconstructing the
original molecule from those fragments, like reassembling a book from millions
of shredded sentences.

**The measurement:** Sequencing (Illumina for short accurate reads, PacBio HiFi
for long accurate reads, ONT for very long reads with higher error).

**The computational translation:**
- Overlap-layout-consensus (OLC): find which reads overlap, build a graph, find
  the path
- de Bruijn graph: break reads into k-mers, find the Eulerian path
- The result is a set of contigs (continuous sequences) — not necessarily
  chromosomes

**Biological validation:**
- Is N50 reasonable for expected genome size and repeat content? A human assembly
  with N50 of 1 kb is fragmented; N50 of 50 Mb is chromosome-scale.
- Are BUSCO scores high? BUSCO checks that conserved single-copy genes expected
  in this lineage are present and complete. < 90% completeness for a model
  organism is a red flag.
- Does assembly size match flow cytometry / k-mer estimated genome size?
- Are telomere sequences present at contig ends (for chromosome-level assemblies)?

**Common misconception:** A contig is not a chromosome. An assembly with 50,000
contigs may have assembled most of the sequence but told you nothing about
chromosome structure.

---

## 2. Read Alignment / Mapping

**The biological story:** After sequencing, each read is a fragment from some
position in the genome. Alignment finds where each fragment came from — it's
like sorting a pile of puzzle pieces back to their positions.

**The measurement:** FASTQ files — each read is a sequence + quality scores.

**The computational translation:**
- Seed-and-extend: find short exact matches (seeds) in an index, extend into full
  alignments
- BWA-MEM, minimap2, STAR are all doing this with different optimisations
- Output: SAM/BAM — each read gets a position, orientation, mapping quality score

**Biological validation:**
- Is the overall mapping rate reasonable? DNA-seq against a good reference: > 95%.
  RNA-seq: 70–85% is normal (splicing is hard). Hi-C: lower is expected (chimeric
  reads, restriction sites).
- Is MAPQ distribution sensible? Many MAPQ=0 reads = lots of multi-mappers =
  either repetitive genome or wrong reference.
- Does coverage look even across chromosomes? Sharp drops suggest reference gaps
  or sample contamination.
- Do known regions (e.g. mitochondria) show the expected depth?

**Common misconception:** High mapping rate ≠ correct mapping. Reads from a
contaminating organism (e.g. mouse from a mouse-passaged human cell line) will
map to the human genome at wrong positions with lower confidence scores.

---

## 3. Variant Calling

**The biological story:** Every individual's genome differs slightly from the
reference (SNPs, indels, structural variants). Variant calling finds those
differences — it's the computational equivalent of finding every typo in a
document compared to a master copy, when you have thousands of imperfect copies.

**The measurement:** Aligned reads (BAM). Variants are positions where the
reads consistently disagree with the reference.

**The computational translation:**
- Pileup: for each position, count the bases observed across all reads
- Statistical model: are the observations consistent with a true variant, or
  random sequencing error?
- GATK HaplotypeCaller, DeepVariant, bcftools call — all do this differently
- Output: VCF file — each variant is a position, reference allele, alt allele,
  genotype, quality

**Biological validation:**
- Ti/Tv ratio: transitions (A↔G, C↔T) should outnumber transversions (~2.1 for
  whole genome, ~3.0 for exome) because transitions are more common biologically
- dbSNP overlap: known variants in databases should account for most of your
  calls in a healthy human sample; a large fraction of novel variants warrants
  scrutiny
- Variant count in context: a healthy human genome has ~4M SNPs vs reference.
  1M = probably under-called. 20M = probably noise.
- Functional annotation: are coding variants in genes whose loss makes biological
  sense for your sample?

**Common misconception:** A variant "called" at 99% confidence is not 99%
certain to exist. It means the model is 99% confident *given* its assumptions.
Alignment errors, PCR duplicates not removed, and reference errors can all
produce high-confidence false positives.

---

## 4. RNA-seq / Differential Expression

**The biological story:** Gene expression is how a cell controls which proteins
it makes. RNA-seq measures the abundance of mRNA molecules in a sample — a
snapshot of transcriptional activity at the moment of cell lysis. Differential
expression asks: which genes are more or less active in condition A vs condition B?

**The measurement:** mRNA → cDNA → sequencing fragments. The number of reads
mapping to a gene is a proxy for mRNA abundance.

**The computational translation:**
- Align reads to genome (STAR, HISAT2) or quantify directly (Salmon, kallisto)
- Count reads per gene (featureCounts, HTSeq)
- Normalise for library size and gene length (RPKM, TPM, or raw counts for DESeq2)
- Statistical test for differential expression (DESeq2, edgeR, limma)

**Biological validation:**
- Are housekeeping genes (ACTB, GAPDH, RPS genes) expressed at similar levels
  across samples? Big differences suggest normalisation problems.
- Are tissue-specific markers present if you know the tissue?
- Does the direction of your top hits make sense? If you knocked down a
  transcription factor, its target genes should be downregulated.
- Does PCA separate your samples by condition, not by batch or processing date?
  If your samples cluster by extraction date, you have a batch effect.
- Volcano plot sanity check: are there genes changing 1000-fold? That's usually
  artefact (pseudogene, multi-mapper issue) not biology.

**Common misconception:** TPM/RPKM normalisation lets you compare genes *within*
a sample, not *between* samples. For between-sample DE, use raw counts fed into
DESeq2 or edgeR which apply their own normalisation.

---

## 5. ChIP-seq / ATAC-seq (Epigenomics)

**The biological story:** Not all DNA is accessible or bound by proteins equally.
ChIP-seq uses antibodies to pull down DNA bound by a specific protein
(transcription factor, histone mark). ATAC-seq uses a transposase to tag open
chromatin. Both ask: *where* in the genome does a given regulatory event occur?

**The computational translation:**
- Align reads (as per normal alignment)
- Call peaks: find genomic regions enriched for reads compared to background
  (MACS2, HOMER)
- Annotate peaks: which genes are nearby? What motifs are in the peak sequences?

**Biological validation:**
- Are peaks at known binding sites for this factor? If you ChIPed for CTCF,
  there should be thousands of peaks genome-wide at CTCF motifs.
- FRiP score (fraction of reads in peaks): > 1–5% for TFs, > 20% for histones.
  Low FRiP = noisy experiment, poor IP efficiency.
- Do peaks overlap with known regulatory databases (ENCODE, Roadmap)?
- For ATAC-seq: is nucleosome positioning visible in fragment size distribution?
  You should see a clear mono-nucleosome peak at ~200 bp and sub-nucleosomal
  at ~100 bp.

---

## 6. Metagenomics / 16S Amplicon

**The biological story:** A microbial community sample (gut, soil, ocean) contains
hundreds of species mixed together. Metagenomics sequences the whole community
("shotgun"), while 16S amplicon sequencing targets a specific marker gene to
identify who is present.

**The computational translation:**
- 16S: cluster reads into OTUs or ASVs by sequence similarity, assign taxonomy
  (QIIME2, DADA2)
- Shotgun: assemble metagenome or align to reference databases, classify reads
  by taxonomy (Kraken2, MetaPhlAn)

**Biological validation:**
- Does alpha diversity match expected complexity for this environment? Gut: high.
  Hospital surface: moderate. Hot spring: low.
- Are the dominant taxa biologically expected? Gut = Firmicutes/Bacteroidetes
  dominated. Ocean = Proteobacteria. Soil = mixed.
- Are there contaminants? Human DNA, kit reagents, water controls.
- Is there a negative control? Sequencing "empty" kits reveals reagent contamination
  that can confound low-biomass samples.

---

## 7. Bacterial Pangenomics

**The biological story:** Two strains of the same bacterial species can differ
by 10–30% of their gene content. The *pan-genome* is the union of all genes
found across strains. The *core genome* is what every strain shares (essential
functions). The *accessory genome* is what only some strains carry — virulence
factors, antibiotic resistance, metabolic flexibility. Understanding which genes
are core vs accessory is central to understanding bacterial ecology and evolution.

**Key concepts:**
- Open pan-genome: adding new strains keeps adding new genes (typical for
  free-living bacteria like *E. coli*)
- Closed pan-genome: new strains add few new genes (typical for host-restricted
  bacteria like *Mycoplasma*)
- Horizontal gene transfer (HGT): bacteria swap genes across lineages — the
  accessory genome is its primary vehicle

**The computational translation:**
- Annotate genomes (Prokka, PGAP)
- Cluster genes by homology (Panaroo, Roary, PPanGGOLiN)
- Classify as core / soft-core / shell / cloud based on frequency across strains
- Analyse pan-genome openness (Heaps' law exponent)

**Biological validation:**
- Is core genome size sensible for the species? *E. coli* core is ~2,000–3,000
  genes; total genome ~4,500–5,500.
- Do accessory genes include known mobile elements (prophages, plasmid remnants,
  insertion sequences)? If not, your clustering may be too stringent.
- Are known virulence or resistance genes in the accessory genome? This is
  expected — they confer fitness advantages in specific niches.
- Does pan-genome openness match ecological lifestyle? Free-living = open;
  obligate intracellular = closed.

---

## 8. Genome-Scale Metabolic Modelling (GEMs)

**The biological story:** Every cell's metabolism is a network of biochemical
reactions, each catalysed by an enzyme encoded by a gene. A GEM maps the
genome to a metabolic network: which reactions can this organism perform, given
its genes? Flux balance analysis (FBA) then asks: given constraints (nutrients
available, thermodynamics), what metabolic fluxes are feasible?

**The computational translation:**
- Draft GEM from genome annotation (CarveMe, ModelSEED)
- Gap-fill to ensure connectivity and growth
- Set constraints (exchange reactions = available nutrients)
- Solve linear programme: maximise growth rate subject to constraints (COBRApy)

**Biological validation:**
- Can the model grow on minimal media that the organism grows on in the lab?
  If not, there are gaps in the model or the wrong medium constraints.
- Does predicted growth rate correlate with measured growth rates across conditions?
- Are essentiality predictions (gene knockouts that prevent growth) consistent
  with experimental data?
- For pangenome GEMs: do accessory metabolic genes predict condition-specific
  growth differences between strains?

**The key limitation:** A single-reference GEM misses accessory metabolism.
Strains with unique biosynthetic clusters (secondary metabolites, novel
degradation pathways) are invisible in core-genome-only models.

---

## Quick Benchmarks by Domain

| Domain | "Normal" value | Red flag |
|---|---|---|
| Mapping rate (DNA-seq, good reference) | > 95% | < 70% |
| Mapping rate (RNA-seq) | 70–90% | < 50% |
| BUSCO completeness | > 95% | < 85% |
| Unique read % (RNA-seq after dedup) | > 60% | < 40% |
| Ti/Tv ratio (WGS SNPs) | ~2.1 | < 1.5 or > 3.0 |
| FRiP (ChIP-seq TF) | > 1% | < 0.5% |
| FRiP (ChIP-seq histone mark) | > 20% | < 5% |
| DE gene count (typical experiment) | 100–5,000 | > 20,000 (suspect) |
| Bacterial core genome fraction | 40–70% | < 20% (over-fragmented assemblies) |
