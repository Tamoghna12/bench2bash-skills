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

**Deeper dive — types of variants:**
- SNPs/indels: point changes and small insertions/deletions. Detected by pileup-based callers (GATK HaplotypeCaller, DeepVariant, bcftools call).
- Structural variants (SVs): deletions, duplications, inversions, translocations >50 bp. Require split-read and read-pair evidence. Tools: Manta, DELLY, LUMPY/SMOOVE, Sniffles (long-read).
- Copy number variants (CNVs): regions duplicated or deleted across many kb–Mb. Tools: CNVnator, GATK gCNV, Control-FREEC.
- Key distinction: hard filtering (apply fixed thresholds to QUAL/DP/GQ fields) vs VQSR soft filtering (train a model on known variants, apply a score). VQSR requires large cohorts; hard filtering is appropriate for small studies.
- Phasing: determining which alleles are on the same haplotype. Relevant for compound heterozygotes in clinical genetics, and for assembly haplotype validation. Tools: WhatsHap, SHAPEIT5.

**Variant annotation pipeline:**
After calling variants, interpretation requires annotation:
1. Effect prediction: SnpEff or VEP (Ensembl) — predicts whether a variant is synonymous, missense, nonsense, splice-site, intergenic, etc.
2. Population frequency: gnomAD / ExAC allele frequencies — a variant present in 5% of the healthy population is likely benign for a rare disease.
3. Conservation scores: PhyloP, GERP, CADD — cross-species conservation is a proxy for functional importance.
4. Clinical databases: ClinVar (clinical significance), OMIM (gene-disease), PharmGKB (pharmacogenomics).

Biological validation addition:
- Allele balance: for a true heterozygous SNP, expect ~50% alt reads. Allele balance of 10% or 90% suggests a false positive or somatic variant.
- Strand bias: a variant called mostly on forward reads (or mostly reverse) is likely a sequencing artefact. Check Fisher strand (FS) and strand odds ratio (SOR) in GATK output.

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

**Library preparation — what it changes:**
- Stranded vs unstranded: stranded protocols (dUTP, NEBNext Directional) preserve which DNA strand the original mRNA came from. Unstranded data cannot distinguish overlapping antisense transcripts. Always check strandedness before running featureCounts/HTSeq (--stranded parameter matters).
- Poly-A selection vs ribo-depletion: poly-A captures mRNA but misses non-polyadenylated transcripts (lncRNA, histone mRNAs, primary transcripts). Ribo-depletion captures everything including pre-mRNA and non-coding RNA.
- Read length and depth: 50 bp reads sufficient for gene-level quantification; 150 bp reads needed for isoform-level analysis; 50M reads/sample typical for DE; 200M+ for rare transcripts.

**Alignment-based vs pseudoalignment:**
- STAR/HISAT2: true alignment to genome; counts spliced reads at exon-exon junctions; slow but produces a BAM you can inspect and reuse.
- Salmon/kallisto: pseudoalignment to transcript sequences; counts are transcript-level (can be aggregated to gene with tximport); 10-50× faster; better isoform quantification; no BAM output.
- When to use Salmon: large datasets, isoform-level questions, speed is a priority.
- When to use STAR: you also need the BAM (for splicing analysis, chimeric reads, variant calling from RNA); you need splice junction counts; you're doing novel transcript discovery.

**Batch effects:**
A batch effect is technical variation from processing date, reagent lot, or sequencer lane that mimics biological signal. Signs: PCA separates samples by date, not condition.
- Remove with: limma::removeBatchEffect (when groups are balanced across batches), ComBat (empirical Bayes), or include batch as a covariate in the DE model.
- The wrong fix: remove batch effect from counts before DE analysis using a simple subtraction — this violates the statistical model. Always model batch as a covariate in DESeq2/edgeR, or use removeBatchEffect only for visualization.
- Critical rule: if all samples from condition A were processed on date 1 and all from condition B on date 2, the batch effect is perfectly confounded with the biological effect. No statistical method can save this experiment.

**Common misconception (expanded):** TPM and RPKM/FPKM are for within-sample comparisons only. To compare expression levels of the same gene across samples, use raw counts with DESeq2/edgeR normalization, or Salmon's posterior counts with tximport.

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

**Input controls (ChIP-seq only):**
Every ChIP-seq experiment requires an input control: DNA from the same cells without immunoprecipitation. The input captures background chromatin accessibility and GC bias. Peak callers (MACS2, HOMER) model the control to compute enrichment above background.
- Without an input control, open chromatin regions will appear as false peaks (ATAC-seq does not need input because it directly measures accessibility).
- Matched input: same cell number, same sonication conditions, same sequencing depth as the ChIP sample.

**MACS2 key parameters:**
- `-g`: effective genome size (hs = human, mm = mouse, or specify in bp). Critical for p-value calculation.
- `--nomodel`: use if fragment size cannot be estimated (very narrow peaks, transcription factors). Specify `--extsize` manually.
- `-q 0.05` vs `-p 0.01`: q-value (FDR) is more conservative than p-value; use q-value for broad/noisy marks.
- Broad peaks (`--broad`): use for histone marks (H3K27me3, H3K9me3, H3K36me3) where signal is distributed over kb; narrow peaks for TFs.

**IDR (Irreproducibility Discovery Rate):**
IDR is the standard reproducibility filter for ChIP-seq and ATAC-seq peaks, used by ENCODE. It compares peak ranks across biological replicates — reproducible peaks rank consistently; irreproducible ones rank randomly.
- Run peak calling on each replicate independently, then on the pooled data.
- Use IDR to find the threshold where replicate concordance is acceptable (IDR < 0.05 = reproducible).
- Peaks that only appear in one replicate should be discarded.

**Motif analysis:**
After peak calling, motif enrichment asks: what transcription factor binding sequences are enriched in your peaks?
- HOMER findMotifsGenome.pl: de novo motif discovery + known motif enrichment against JASPAR/TRANSFAC databases.
- MEME-ChIP: web-based suite for de novo and enrichment analysis.
- fimo: scan sequences for specific known motifs.
- Key interpretation: finding the expected TF motif in your ChIP peaks is a positive control. Unexpected dominant motifs often indicate co-precipitated TFs (co-factors, or a contaminating antibody cross-reactivity).

**Blacklist regions:**
Some genomic regions (centromeres, telomeres, rDNA arrays, mitochondrial pseudo-nuclear insertions) produce artefactual signal in virtually all ChIP-seq and ATAC-seq experiments due to high copy number and repetitive sequence. Always filter peaks against the ENCODE blacklist before analysis:
```bash
bedtools intersect -v -a peaks.bed -b ENCFF356LFX.bed > peaks.filtered.bed
```
ENCODE blacklists are available for hg38, hg19, mm10, dm6, ce10, danRer10.

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

**OTU vs ASV — why the field moved:**
- OTUs (Operational Taxonomic Units): sequences clustered at 97% identity. Fast but loses resolution; sequences from different species can be merged; sequencing errors can create phantom OTUs.
- ASVs (Amplicon Sequence Variants): exact sequences after error correction (DADA2, Deblur). Higher resolution; reproducible across studies (same ASV = same sequence); better for detecting closely related strains. ASVs are now the standard; use OTUs only when comparing to legacy datasets that used them.

**Taxonomic databases matter:**
- SILVA (16S/18S/ITS): gold standard for 16S assignment, but version matters enormously. SILVA 138 and SILVA 132 give different taxonomies for some lineages. Always specify the version used.
- GTDB (Genome Taxonomy Database): prokaryote taxonomy based on phylogenomics, not 16S. More consistent than NCBI/SILVA for bacterial taxonomy but incompatible with older literature.
- MetaPhlAn uses clade-specific marker genes (not 16S) from its own curated database (mpa_vJan21 etc.). Version must match the database.

**Statistical considerations for microbiome data:**
- Compositionality: 16S data are proportional (sum to 1 per sample). Standard statistical tests assume independence of features, which is violated. Use log-ratio transforms (ALR, CLR) or tools designed for compositional data (ANCOM, ANCOM-BC, ALDEx2).
- Rarefaction: subsampling all samples to the same depth before alpha diversity analysis. Still debated; it discards data but equalises sampling effort. Alternatives: size-factor normalization (DESeq2-style, via metagenomeSeq or ANCOM-BC) or rarefying curves to check saturation.
- Beta diversity tests: PERMANOVA (vegan::adonis2) tests whether groups differ in composition. Always check homogeneity of dispersions (betadisper) — PERMANOVA is sensitive to differences in variance, not just centroid location.

**Functional annotation (shotgun metagenomics):**
HUMAnN3 (HMP Unified Metabolic Analysis Network 3) maps metagenomic reads to functional pathways:
1. Aligns reads to a pangenome database (ChocoPhlAn) to identify which organisms are present
2. Maps the remaining unclassified reads to a protein database (UniRef90)
3. Quantifies pathway abundance and coverage

Output: pathway abundance table (RPKM) and coverage table per sample. Use for testing whether functional pathways differ between groups even when taxonomy is confounded.

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

**Species boundary and ANI:**
Average Nucleotide Identity (ANI) is the de facto standard for prokaryotic species delineation. The 95% ANI cutoff corresponds roughly to the traditional 70% DDH (DNA-DNA hybridization) threshold.
- Strains with ANI > 95%: same species (include in pan-genome analysis together)
- Strains with ANI 85–95%: same genus, may be separate species
- Tools: fastANI (fast, good for large datasets), PyANI (pairwise, with statistical testing)
- Pitfall: pan-genome tools assume all input genomes are the same species. Including strains below the ANI cutoff inflates the accessory genome and produces spurious "core" genes.

**Recombination detection:**
Bacterial genomes exchange DNA via horizontal gene transfer (HGT), which confounds phylogenetic inference. Before building a core-genome phylogeny, detect and mask recombinant regions:
- ClonalFrameML: maximum likelihood, reconstructs clonal frame and identifies recombinant imports
- Gubbins (Genealogies Unbiased By recomBinations In Nucleotide Sequences): faster for large datasets, identifies recombinant blocks from spatial clustering of SNPs
- phi test (PhiPack): simple permutation test for recombination signal in an alignment

After masking recombination, the core-genome SNP phylogeny is more reliable for inferring vertical evolutionary relationships.

**Gene calling quality affects pan-genome:**
Pan-genome clustering tools depend on accurate gene calls. Prokka (Prodigal + HMMER) and PGAP (NCBI annotation pipeline) can call different gene numbers for the same genome. Fragmented assemblies produce truncated genes that look like novel accessory genes. Best practice:
- Use complete (chromosome-level) assemblies where possible
- Re-annotate all genomes with the same tool and parameters for consistency
- Panaroo is more robust to annotation inconsistencies than Roary (it uses a graph-based approach that handles gene fragmentation and edge cases)

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

**Medium composition — the most important parameter:**
In FBA, the growth medium is encoded as exchange reactions with defined bounds. Setting exchange flux incorrectly (e.g. allowing unlimited glucose in a carbon-limited experiment) produces meaningless growth predictions.
- Default media in CarveMe/ModelSEED are often too permissive. Always constrain to the actual experimental medium.
- Exchange reaction: EX_glc__D_e (glucose exchange in BiGG notation). Set lower bound to -10 mmol/gDW/h for carbon-limited growth; set to 0 to block a nutrient.
- Validate: can the model grow on rich medium (LB/BHI)? On minimal medium (M9 + glucose)? These are the simplest sanity checks.

**Gap-filling pitfalls:**
Gap-filling algorithms add reactions to make the model grow, but they can add biologically impossible reactions (reactions from distantly related organisms, thermodynamically unfavorable reactions, or shortcuts that bypass regulatory steps).
- Validate gap-filled reactions against literature: does the organism actually have this pathway?
- Prefer gap-fill solutions with fewer reactions (penalise reaction additions in the objective)
- Check if the gap-filled reaction has a known gene in the organism's genome — if no gene can be identified, the reaction is suspicious

**When FBA fails to reflect reality:**
FBA assumes: (1) steady-state flux distribution, (2) growth rate maximisation, (3) no kinetics. These assumptions fail when:
- The organism is in exponential phase but not maximising growth (stationary phase, stress response)
- Regulatory constraints prevent optimal flux (e.g. carbon catabolite repression)
- You want to model batch dynamics (use dynamic FBA, dFBA, instead)
For time-course or non-steady-state questions, consider kinetic models (ODE-based) or resource allocation models (ME-models).

---

## 9. Single-Cell Sequencing (scRNA-seq / scATAC-seq)

**The biological story:** Bulk RNA-seq measures the average expression across all cells in a sample, masking the heterogeneity between individual cells. Single-cell sequencing sequences each cell independently, revealing distinct cell types, states, and developmental trajectories within a tissue. A tumour biopsy that bulk RNA-seq treats as one sample may contain cancer cells, immune infiltrates, fibroblasts, and endothelial cells — each with completely different expression programmes.

**The measurement:** Cells are isolated (droplet-based: 10x Genomics, Parse Biosciences; plate-based: Smart-seq2), lysed, and barcoded so every read is tagged with the cell of origin. Each cell also gets a Unique Molecular Identifier (UMI) per transcript to count molecules rather than reads (corrects for PCR amplification bias).

**The computational translation:**
1. **Alignment and counting:** STARsolo, Cell Ranger (10x), or Salmon/alevin produces a cell × gene count matrix. Rows = cells; columns = genes; values = UMI counts.
2. **Quality filtering:** Remove low-quality cells based on:
   - nFeatures (genes detected): too low = empty droplet or dead cell; too high = doublet
   - nCounts (total UMIs): very low = poor capture; very high = doublet
   - % mitochondrial reads: high % mito = dying cell (mitochondrial RNA is cytoplasmic and persists after cell lysis)
   - Typical thresholds: nFeatures 200–6000, %mito < 20% (tissue-dependent)
3. **Normalisation:** Divide each cell by its total counts and log-transform (log1p(count/total × 10,000)). This corrects for library size differences between cells. SCTransform (Seurat) uses a regularised negative binomial regression — handles highly variable genes more robustly.
4. **Dimensionality reduction:**
   - PCA on highly variable genes first (reduces from ~20,000 genes to 30–50 PCs)
   - UMAP or t-SNE on top PCs for 2D visualisation
   - UMAP preserves more global structure than t-SNE; t-SNE is better for tight local cluster separation
5. **Clustering:** Graph-based clustering (Leiden or Louvain algorithm) on the PCA neighbourhood graph. Resolution parameter controls granularity — higher resolution = more, smaller clusters.
6. **Marker genes and cell type annotation:**
   - Find differentially expressed genes per cluster (Wilcoxon rank-sum, MAST, or edgeR)
   - Annotate clusters by known marker genes (CD3D = T cells, CD14 = monocytes, EPCAM = epithelial)
   - Automated annotation: SingleR (reference-based), CellAssign, Azimuth
7. **Doublet detection:** Remove cells that are likely two cells captured together: Scrublet, DoubletFinder. Doublets appear as intermediate cell types with high UMI counts.

**Biological validation:**
- Does UMAP show expected cell types for the tissue? A gut epithelium should have enterocytes, goblet cells, enteroendocrine cells, stem cells.
- Are known marker genes specific to their expected clusters?
- Does the proportion of cell types make biological sense? A tumour with 80% immune cells and 20% cancer cells is biologically plausible; 80% neurons in a liver biopsy is not.
- Are batch effects removed (if applicable)? Check UMAP: cells from different samples or batches should mix within clusters. If they don't, use Harmony, Seurat integration, or scVI for batch correction.

**scATAC-seq — the open chromatin version:**
scATAC-seq sequences accessible chromatin per cell (using the same transposase approach as bulk ATAC-seq). The data matrix is cells × genomic peaks instead of cells × genes. Analysis follows the same general flow but uses ArchR or Signac (Seurat extension) rather than Seurat. Key validation: fragment size distribution should show nucleosomal banding (sub-nucleosomal ~200 bp, mono-nucleosomal ~400 bp).

**Common misconception:** UMAP cluster positions are not quantitative. The distance between clusters in UMAP space is not proportional to biological similarity — UMAP is a visualisation tool, not a measurement. Biological conclusions should come from differential expression or trajectory analysis, not visual UMAP distance.

**Tools:**
- Preprocessing: Cell Ranger (10x), STARsolo, alevin-fry (fast)
- Analysis: Seurat (R), Scanpy (Python), Bioconductor (SingleCellExperiment)
- Batch correction: Harmony, scVI, Seurat RPCA integration
- Trajectory: Monocle3, PAGA, scVelo (RNA velocity)
- scATAC: ArchR, Signac

---

## Quick Benchmarks by Domain

| Domain | Metric | Normal range | Red flag |
|--------|--------|--------------|----------|
| DNA-seq alignment | Mapping rate (good reference) | > 95% | < 70% |
| RNA-seq alignment | Mapping rate | 70–90% | < 50% |
| Hi-C alignment | Mapping rate | 70–90% | < 60% |
| Genome assembly | BUSCO completeness | > 95% | < 85% |
| Genome assembly | NG50 vs expected | Close to expected chromosome size | < 1 Mb for mammal (fragmented) |
| Genome assembly | Assembly size vs genome size estimate | ±10% of GenomeScope estimate | > 20% discrepancy |
| RNA-seq | Unique read % (after dedup) | > 60% | < 40% (high PCR duplication) |
| RNA-seq | DE gene count (typical 2-condition) | 100–5,000 | > 20,000 (likely batch effect or normalisation issue) |
| Variant calling | Ti/Tv ratio (WGS SNPs) | ~2.1 | < 1.5 or > 3.0 |
| Variant calling | Known variant (dbSNP) overlap (human) | > 85% | < 70% (noisy calling) |
| Variant calling | Total SNP count (human WGS) | ~4 million | < 1M (under-called) or > 10M (contamination/error) |
| ChIP-seq | FRiP (TF peaks) | > 1% | < 0.5% |
| ChIP-seq | FRiP (histone mark) | > 20% | < 5% |
| ChIP-seq | Peak count (TF, genome-wide) | 1,000–50,000 | < 100 (poor IP) or > 200,000 (very low stringency) |
| ATAC-seq | Fragment size nucleosomal banding | Clear mono/di-nucleosome peaks | Flat distribution (poor library) |
| ATAC-seq | FRiP | > 20% | < 10% |
| 16S amplicon | Alpha diversity (Shannon) | Gut: 3–5; Soil: 5–8 | Gut < 1.5 (dysbiosis or poor library) |
| scRNA-seq | Genes per cell (human) | 1,000–5,000 | < 200 (empty droplets) or > 8,000 (doublets) |
| scRNA-seq | Median UMIs per cell | 500–5,000 | < 100 (poor capture) |
| scRNA-seq | % mitochondrial reads | < 20% | > 30% (dying cells) |
| Bacterial pan-genome | Core genome fraction | 40–70% | < 20% (over-fragmented assemblies or mixed species) |
| Metabolic model | Growth rate on rich medium | > 0 (model grows) | 0 = ungrowable (critical gap in model) |
