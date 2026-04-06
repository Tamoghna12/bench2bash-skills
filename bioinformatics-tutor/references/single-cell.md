# Single-Cell Sequencing — Teaching Reference

For each domain: the biological question it answers, the measurement, the
computational translation, how to validate the result biologically, and the
misconceptions that cause the most damage in practice.

---

## Why Single-Cell Changes Everything

**The averaging problem.** Bulk RNA-seq gives you the mean gene expression of
however many cells are in your sample — often millions. That average is
biologically meaningful if your sample is homogeneous (a pure cell line, a
bacterial culture in log phase). It is actively misleading if your sample is a
mixture. A tumour biopsy contains cancer cells, fibroblasts, endothelial cells,
immune infiltrate, and normal epithelium. Bulk RNA-seq reports a weighted average
across all of them. A gene that is highly expressed in 5% of cells and silent in
the rest can look moderately expressed in bulk — and you will never know it is
restricted to a rare population.

**Cell type heterogeneity vs cell state variation.** These are different problems.
Cell type heterogeneity is the co-existence of fundamentally distinct cell
lineages in the same tissue — hepatocytes and Kupffer cells are not the same cell
type, and treating them as one in bulk is a category error. Cell state variation
is the continuous spectrum of activation states within one cell type — a
macrophage responding to LPS vs a resting macrophage are the same cell type in
different states. Single-cell resolves both. Bulk resolves neither.

**When single-cell is the right tool:**
- Characterising tissue composition or cell type abundance changes between
  conditions (e.g. tumour vs normal, treated vs untreated)
- Identifying rare cell populations (< 1% of cells) whose biology is masked
  in bulk
- Ordering cells along a developmental or differentiation continuum
- Discovering novel cell types or states not in any reference atlas
- Resolving which cell types drive a bulk DE signal

**When bulk is better:**
- The biological question is about the aggregate response of a defined population
  (e.g. how does this signalling pathway respond in sorted T cells?)
- You need high power for subtle effects: bulk has deeper coverage per gene and
  better statistical power at moderate effect sizes
- You need isoform-level resolution: full-length plate-based scRNA-seq covers
  this partially, but bulk with long reads is still superior
- Cost and throughput: bulk is 10–50× cheaper per sample at comparable depth

The single-cell vs bulk decision should be made by the biology, not by what is
fashionable.

---

## 1. scRNA-seq — Gene Expression at Single-Cell Resolution

### The Biological Story

Every cell in an organism carries (nearly) the same genome. Different cell types
exist because different genes are expressed — transcription factors activate and
repress gene programmes, epigenetic marks lock in or exclude accessibility, and
signalling history shapes which genes are currently transcribed. The result is
that a hepatocyte and a neuron, with identical DNA, look and behave completely
differently because they express different subsets of that DNA.

scRNA-seq measures this: the mRNA molecules present in a single cell at the
moment it is lysed. This is a snapshot, not a movie. A cell that is midway through
differentiating looks exactly like a cell that is midway through differentiating —
you cannot tell, from a single time point, whether it arrived there from a
progenitor or is about to become something else. This is the fundamental
limitation of cross-sectional single-cell data, and it must be understood before
trajectory analysis (Section 5) is attempted.

### The Measurement

**Droplet-based (10x Chromium, Parse Biosciences, BD Rhapsody):**

Cells are suspended in solution and flowed through a microfluidic chip alongside
gel beads coated with oligonucleotides. Ideally, one cell and one bead co-encapsulate
in one aqueous droplet in oil. Inside the droplet, the cell is lysed, mRNA
molecules are captured by the oligos on the bead (via poly-A hybridisation), and
a reverse transcription reaction converts mRNA to cDNA. Each bead carries a
unique *cell barcode* — a short DNA sequence that will label every cDNA molecule
from that cell. Each mRNA molecule gets a unique *UMI (Unique Molecular
Identifier)* — a random short sequence that tags one capture event.

After emulsion breakage, all cDNA is pooled and amplified by PCR, then sequenced.
The cell barcode tells you *which cell* a read came from. The UMI tells you *which
original mRNA molecule* that read derived from.

**Why UMIs are essential:** PCR amplification is stochastic. One mRNA molecule
may generate 5 PCR copies; another may generate 500. Without UMIs, read counts
would reflect PCR efficiency as much as mRNA abundance. With UMIs, you count
*distinct molecules* (distinct barcode + UMI combinations), not reads. UMI
deduplication removes amplification bias entirely, leaving a count that is
proportional to the number of captured mRNA molecules.

**Cell capture efficiency:** 10x Chromium captures approximately 65% of cells in
the input suspension. The rest are lost (not captured, or captured in empty
droplets). This is a fundamental limit — you cannot recover what was not captured.
Every cell barcode in the data corresponds to a droplet, not necessarily a cell.
Many droplets are empty but contain ambient RNA from the solution (see Section 3).

**10x throughput and depth:** Standard 10x captures 500–10,000 cells per lane,
detects ~1,000–3,000 genes per cell, and sequences at ~20,000–50,000 reads per
cell. Gene detection is sparse: typically only 10–20% of expressed genes are
detected in any given cell. This is dropout, not silence — the gene may be
expressed but its mRNA was not captured.

**Plate-based (Smart-seq2, Smart-seq3):**

Individual cells are sorted into wells of a 96- or 384-well plate (by FACS or
micromanipulation). Each well receives a lysis buffer and reverse transcription
reagents. cDNA from each cell is amplified independently and sequenced as a
separate library. There are no cell barcodes (each library IS one cell) and
historically no UMIs (Smart-seq2), though Smart-seq3 added UMIs.

Advantages: full-length transcript coverage (you can detect splice variants),
deeper coverage per cell (~10,000 genes detected vs ~2,000 for 10x), and much
better sensitivity for rare or lowly expressed transcripts.

Disadvantages: expensive and slow (hundreds of cells per plate, not thousands),
requires FACS sorting (you must already know what you're selecting for), and
not compatible with very small cells that are hard to sort.

Use plate-based when you need isoform resolution, deep coverage, or are working
with a specific rare population you can purify by a surface marker.

### The Computational Translation

**Step 1 — Alignment and quantification**

Raw FASTQ files from a droplet experiment contain two read files per sample: one
read carries the cell barcode + UMI, the other carries the cDNA insert (the
transcriptomic read). Alignment tools handle this asymmetry.

- **Cell Ranger** (10x Genomics): proprietary but easy to use. Aligns with STAR,
  calls cells, produces a feature-barcode matrix. The standard for 10x data.
- **STARsolo**: open-source, part of STAR. Matches Cell Ranger output closely.
  Use when you need a reproducible, freely auditable pipeline.
- **alevin-fry** (with simpleaf): salmon-based, extremely fast, lower memory.
  Good for large datasets or resource-constrained environments.
- **kallisto|bustools (kb-python)**: pseudoalignment-based, fastest option.
  Slightly less accurate at multi-mapper resolution but acceptable for most
  downstream analyses.

All of these produce the same output: a cells × genes sparse count matrix where
each entry is the number of UMIs detected for gene g in cell c.

**Step 2 — Cell barcode calling (separating real cells from empty droplets)**

Not every barcode corresponds to a cell. Most correspond to empty droplets with
ambient RNA. The challenge: empty droplets are not empty — they contain free RNA
from the solution, so they produce real counts. Cell barcode calling separates
the two.

The classical method is the *knee plot*: rank barcodes by total UMI count and
plot. Real cells form a plateau at high counts; empty droplets form a long tail at
low counts; the "knee" is the inflection between them. Cell Ranger's default
algorithm finds this knee automatically.

The problem: the knee is sometimes unclear (e.g. in samples with many dying cells,
or very small cells with low RNA content). **EmptyDrops** (implemented in
DropUtils) takes a statistical approach: it models the ambient RNA profile from
the lowest-count barcodes, then tests each barcode for whether its count profile
is significantly different from ambient. This recovers more true cells, especially
small or lowly active populations that sit near the knee.

Rule of thumb: use EmptyDrops for any experiment where you expect cell type
diversity in abundance or RNA content. Use the knee method when you have a clean,
uniform sample.

**Step 3 — Quality filtering (per cell)**

After barcode calling, filter cells on three metrics. Each captures a different
biological failure mode:

- **nFeatures (number of genes detected):** Too few = dying cell, empty droplet
  that slipped through, or a cell type with genuinely low transcriptional activity.
  Too many = almost certainly a doublet (two cells co-captured). Typical range:
  200–6,000 genes. The upper cutoff is dataset-specific; set it based on the
  distribution, not a fixed number.

- **nCounts (total UMIs detected):** Correlated with nFeatures. Very low = poor
  quality. Very high = doublet. Examine this alongside nFeatures; a high nCounts
  with a low nFeatures ratio (few highly expressed genes) might be a contaminated
  droplet, not a real cell.

- **Percent mitochondrial reads (%mito):** Mitochondrial genes encode components
  of the respiratory chain. In a live, intact cell, mitochondrial mRNA is a small
  fraction of total mRNA (~5–20% depending on cell type and metabolic state). In a
  dying or lysed cell, the cytoplasmic mRNA has leaked out but the mitochondria
  (being intact organelles) are retained. This inflates %mito. A high %mito cell
  is usually a dying cell. Common cutoff: remove cells with > 20–25% mito, but
  calibrate per tissue. Cardiac muscle, hepatocytes, and retinal cells have
  genuinely high mitochondrial content and will be incorrectly filtered at low
  cutoffs.

Do NOT apply hard universal cutoffs. Look at the distributions in your data and
set cutoffs that remove clear outliers while preserving genuine biological
heterogeneity.

**Step 4 — Normalisation**

After filtering, each cell has a different total UMI count (library size). This
is technical, not biological — it reflects capture efficiency and sequencing depth,
not how much RNA the cell actually contained. Normalisation makes cells comparable.

- **Log-normalisation (Seurat NormalizeData, Scanpy normalize_total + log1p):**
  Divide each cell's counts by a size factor (total counts × scaling factor, e.g.
  10,000), then log-transform. Fast, interpretable, and the default for most
  analyses. The assumption: total counts is a good proxy for library size. This is
  approximately true for most cells but breaks down when cell types differ
  dramatically in total RNA content.

- **SCTransform (Seurat) / scran pooling-based normalisation:** Fits a regularised
  negative binomial model per gene (SCTransform) or uses pooled size factors
  (scran). More principled statistically — accounts for the mean-variance
  relationship in count data. Better for datasets with high technical noise or
  mixed cell types that differ greatly in RNA content. More computationally
  intensive. Use SCTransform for v2 (Seurat v4) or SCTransform v2 (Seurat v5) for
  improved performance.

When to use which: log-normalisation is fine for exploratory analysis, within a
dataset of relatively uniform cell types. Prefer SCTransform or scran when you
are integrating across batches or have evidence that total RNA content varies
biologically across your cell types.

**Step 5 — Feature selection (highly variable genes)**

A human gene expression matrix has ~20,000–30,000 rows. Most of these genes are
either not expressed at all (zero in most cells) or constitutively expressed at
similar levels everywhere (housekeeping genes). Neither type contains information
about cell type differences. Running PCA or clustering on all genes wastes
computation and dilutes signal with noise.

Highly variable gene (HVG) selection retains genes that are variable *beyond what
is expected from technical noise* at their expression level. The standard approach
(FindVariableFeatures in Seurat, highly_variable_genes in Scanpy) models the
mean-dispersion relationship and selects genes above the trend. Typically 2,000–
5,000 HVGs are selected.

Intuition: housekeeping genes like ACTB or GAPDH are expressed in every cell at
similar levels — they have low variance relative to their mean. A transcription
factor expressed in one cell type but not others will have high variance. HVG
selection preferentially keeps the latter.

**Step 6 — Dimensionality reduction: PCA, UMAP, t-SNE**

The HVG matrix (cells × 2,000–5,000 genes) is still high-dimensional. PCA
reduces this to ~10–50 principal components that capture most of the variance.
Each PC is a linear combination of genes; the top PCs capture the largest axes
of transcriptional variation (often corresponding to major cell type differences).

UMAP or t-SNE then further reduces the PCA embedding to 2D for visualisation.
These are *non-linear* methods that emphasise local structure: cells that are
transcriptionally similar are placed near each other. The key point (see
Misconceptions below): the global geometry of a UMAP is not interpretable.

The pipeline in practice:
1. Scale data (zero-mean, unit-variance per gene)
2. PCA on HVGs (use 30–50 PCs initially)
3. Determine how many PCs to use (elbow plot, JackStraw)
4. Build a KNN graph on the PCA embedding (k typically 10–30)
5. Run UMAP or t-SNE on the graph (not on the raw PCA)
6. Use the PCA embedding (not UMAP) for all quantitative analyses

**Step 7 — Clustering**

Graph-based clustering (Leiden algorithm, Louvain algorithm) partitions the KNN
graph into communities. The *resolution parameter* controls cluster granularity:
higher resolution = more, smaller clusters. There is no objectively correct
resolution. The right resolution is the one where the clusters are biologically
meaningful for your question.

Common mistake: increase resolution until every cluster looks unique, then report
all clusters as distinct cell types. This is overfitting. A cluster is only a cell
type if it has a coherent biological identity supported by marker genes, not just
because the algorithm separated it.

**Step 8 — Marker gene detection and differential expression**

For each cluster vs all others (or pairwise), test which genes are enriched.
Standard methods:
- **Wilcoxon rank-sum test**: non-parametric, robust, default in Seurat and
  Scanpy. Tests whether gene expression ranks differ between groups.
- **MAST**: hurdle model that handles the zero-inflation of scRNA-seq counts by
  modelling the probability of detection and the level of expression separately.
  More statistically principled; recommended when comparing conditions with
  covariates.
- **pseudobulk + DESeq2/edgeR**: see Section 7. This is the correct approach
  for multi-sample comparisons, not single-cell DE.

**Step 9 — Cell type annotation**

Manual annotation: look at marker genes for each cluster and assign identity based
on known biology. CD3D+ CD8A+ = CD8 T cell. EPCAM+ KRT19+ = epithelial cell.
This requires domain knowledge and is the most reliable method when done carefully.

Automated annotation:
- **SingleR**: compares each cell's expression profile to bulk RNA-seq reference
  datasets (Human Primary Cell Atlas, Blueprint, Encode) and assigns the closest
  reference label. Fast, transparent, good for well-defined cell types.
- **Azimuth (Seurat)**: projects your cells onto a large curated reference atlas
  (e.g. Human Cell Atlas PBMC reference). Returns labels and confidence scores.
  Excellent for PBMC and a growing number of tissues with reference atlases.
- Both are reference-limited: if your sample contains a cell type not in the
  reference, it will be misclassified or assigned low confidence. Manual review
  is always necessary.

### Biological Validation

Before trusting any annotation:
- Check that expected cell types are present for the tissue. A liver biopsy
  should contain hepatocytes, Kupffer cells (liver macrophages), stellate cells,
  endothelial cells, and cholangiocytes. If one is absent, ask whether it was
  lost in the protocol or genuinely absent.
- Check known marker gene specificity. CD3D should be in T cells only. PTPRC
  (CD45) should be in all immune cells. EPCAM should be in epithelial cells only.
  If markers bleed across clusters, the clusters are under-separated or there
  is ambient RNA contamination.
- Sanity-check cell type proportions. In normal human PBMC, ~70% are T cells,
  ~10–15% B cells, ~10–15% monocytes, ~5% NK cells, and ~1% dendritic cells.
  A sample that is 80% monocytes from a healthy donor is suspect.
- If proportions differ between samples, ask whether this is biology (disease
  effect on cell composition) or technical (differential capture efficiency,
  batch effect on clustering).

### Common Misconceptions

**UMAP distances are not quantitative.** Two clusters that appear close on UMAP
are not necessarily more transcriptionally similar than two clusters that appear
far apart. UMAP optimises local neighbourhood relationships; the global distances
are arbitrary. Do not say "these cell types are similar because they cluster
together on UMAP." Use the PCA embedding or marker gene overlap for that claim.

**More clusters is not more resolution.** Increasing the clustering resolution
until each cluster is unique does not reveal more biology — it reveals more noise.
The algorithm will always find something to split if asked to. Every new cluster
must be justified by distinct biological identity (unique markers, distinct
functional annotation, validated gene programmes), not just by algorithmic
separability.

**Batch effects in scRNA-seq are harder than in bulk.** In bulk, every sample has
a library size and a global shift in expression that can be corrected. In
scRNA-seq, each cell has its own library size that varies enormously (often 10×
within a sample), and batch effects interact with cell type composition (if one
sample has more of one cell type, the batch effect on that cell type contaminates
the whole sample). Never assume that a shared normalisation step removes batch
effects. Use explicit batch correction (Section 4).

---

## 2. Doublets — The Most Common Silent Error

**What doublets are:** A doublet forms when two cells co-encapsulate in one
droplet. They receive one cell barcode and appear as a single cell in the count
matrix. Their gene expression profile is the sum of the two cells' transcriptomes.

**Why they matter:** Doublets appear in unexpected locations in UMAP space. A
doublet of a T cell and a B cell will have intermediate or mixed expression and
may cluster as a novel "cell type" between the two. Doublets are among the most
common sources of spurious novel populations in single-cell papers.

**Expected doublet rate:** For 10x Chromium, approximately 0.8% of barcodes are
doublets per 1,000 cells targeted. At the standard 5,000 cell target, expect ~4%
doublets. At 10,000 cells, ~8%. This is why loading too many cells increases
doublet rate superlinearly — 10x explicitly trades doublet rate against cell
recovery.

**Detection tools:**
- **Scrublet** (Wolock et al.): simulates synthetic doublets by summing pairs of
  real cell profiles, then scores each real cell by how similar it is to simulated
  doublets. Returns a doublet score per cell. Fast, widely used.
- **DoubletFinder** (McGinnis et al.): similar simulation approach but integrates
  with the Seurat workflow. Requires specifying the expected doublet rate (use the
  0.8%/1000 cells rule). More flexible parameter tuning.
- **scDblFinder** (Germain et al.): classifier-based approach, considered among
  the most accurate in benchmarks (Xi and Li 2021).

Run doublet detection *before* batch correction and *before* clustering. Doublet
scores should be visualised on UMAP and cells with high scores removed. Do not
remove based on score alone — also check that removed cells are not a real
rare population at the intersection of two types.

---

## 3. Ambient RNA Contamination

**What it is:** When cells are processed for single-cell sequencing, some cells
inevitably lyse before encapsulation. Their mRNA is released into the surrounding
solution — the "soup." Every droplet, including empty droplets, contains some of
this ambient RNA. When a cell is captured, its count matrix reflects both its own
transcriptome and a small contribution from ambient RNA.

**Why it matters:** Ambient RNA is not random. It reflects which genes are highly
expressed in the most abundant and most fragile cell types in your sample. In a
PBMC sample, red blood cells are abundant and fragile: haemoglobin genes (HBA1,
HBA2, HBB) will appear in the ambient pool. In a liver sample, hepatocyte genes
are highly expressed and will contaminate non-hepatocyte clusters. The result:
every cluster appears to express marker genes from the dominant cell type, even if
those cells do not actually express them.

**Detection and correction:**
- **SoupX** (Young and Behjati): estimates the ambient RNA profile from empty
  droplets, estimates the contamination fraction per cluster, and corrects the
  count matrix. Requires cluster labels (run after basic clustering). Produces a
  corrected count matrix that is used as the starting point for all downstream
  analysis.
- **DecontX** (Yang et al., celda): Bayesian model that estimates contamination
  per cell. Included in the celda/scran ecosystem.

**How much contamination is typical:** 2–20% of counts in a given cell may be
ambient, depending on tissue processing quality. Freshly dissociated tissue
processed quickly has lower ambient contamination than frozen tissue with a slow
protocol.

**The practical check:** Before running SoupX, look at your marker genes. If
haemoglobin genes appear as markers of non-erythroid clusters, you have ambient
RNA contamination. If EPCAM (epithelial marker) appears in fibroblast clusters,
same issue.

---

## 4. Batch Correction

**Why scRNA-seq is especially vulnerable to batch effects:** Sequencing date,
operator, reagent lot, ambient temperature during cell isolation, and time from
tissue dissociation to encapsulation all introduce technical variation. In
scRNA-seq this manifests as: cells from the same cell type but different batches
cluster separately on UMAP. This is not biology — it is technical variation in
library preparation.

The problem is amplified because scRNA-seq library sizes vary so much per cell
that a global correction (as in ComBat for bulk) is insufficient. Each cell needs
its own correction term, which is estimated jointly from all cells.

**Methods:**

- **Harmony** (Korsunsky et al., 2019): runs iterative soft clustering in PCA
  space, corrects PC coordinates to minimise batch variance within clusters.
  Fast, scalable to millions of cells, integrates seamlessly into Seurat and
  Scanpy workflows. The de facto standard for most analyses. Run on PCA
  embeddings; produces a corrected PCA that is used for UMAP and clustering.

- **Seurat RPCA integration**: reference-based integration using reciprocal PCA.
  More conservative than CCA integration (earlier Seurat method). Better when
  batches share only a subset of cell types.

- **scVI** (Lopez et al., 2018): variational autoencoder that models count data
  directly. Best for very large datasets (>500,000 cells) where Harmony begins to
  struggle, or where batches differ substantially in cell type composition. Slower
  to train, requires GPU for large datasets.

- **BBKNN** (Polanski et al.): batch-aware KNN graph construction. Simple, fast,
  and integrates directly into Scanpy.

Benchmarking (Luecken et al. 2022, scIB): scVI and Harmony perform best overall.
For most labs, Harmony is sufficient and far easier to run.

**How to assess whether correction worked:**
- Plot UMAP coloured by batch before and after correction. After correction,
  cells from different batches should intermix within each cell type cluster.
- Compute integration metrics: local inverse Simpson's Index (LISI) should
  increase (more mixing) after correction, while cell type purity should be
  preserved.
- Check that biologically meaningful differences between samples still exist after
  correction. Over-correction can merge cell types that differ between conditions.

**When NOT to correct for batch:**
- When batch is confounded with biology. If all disease samples were processed on
  one date and all controls on another, batch correction will remove the disease
  signal along with the technical noise. There is no computational fix for a
  confounded experimental design.
- When comparing samples where biological differences between groups are expected
  to drive clustering (e.g. comparing cells from two very different tissues). In
  this case, batch correction may merge the biology.

---

## 5. Trajectory Analysis and RNA Velocity

### Pseudotime

**The biological question:** Cells undergoing differentiation or response to
stimuli exist along a continuum of states. Cross-sectional single-cell data
captures a snapshot of all these states simultaneously. Pseudotime analysis orders
cells along an inferred developmental path — assigning each cell a relative
position along the continuum.

**Tools:**
- **Monocle 3** (Trapnell lab): builds a principal graph through the low-
  dimensional embedding, identifies branch points, and assigns pseudotime from a
  user-specified root. Best for complex topologies (multiple branches, cycles).
- **PAGA** (Wolf et al., Scanpy): Partition-based graph abstraction. Creates an
  abstracted graph of cluster connectivity weighted by evidence for inter-cluster
  transitions. Useful as a first pass to understand the global topology before
  computing per-cell pseudotime.
- **Diffusion pseudotime** (DPT, Haghverdi et al., Scanpy): eigenvector-based
  pseudotime, computationally efficient, good for linear or simple branching
  topologies.

**Critical misconception:** Pseudotime is not real time. A cell assigned
pseudotime = 10 is not older or younger in absolute time than a cell with
pseudotime = 5. It is simply farther along the inferred continuum of transcriptional
change. You cannot convert pseudotime to hours without orthogonal data (e.g.
metabolic labelling, time-course sampling).

The root cell (pseudotime = 0) must be chosen biologically. Monocle 3 will accept
any cell as the root — choosing the wrong root (e.g. a terminal cell type instead
of a progenitor) inverts the trajectory. Always justify root choice with
biological knowledge (the progenitor or stem cell should be the root).

### RNA Velocity

**The biological story:** mRNAs are produced (transcription) and degraded
(turnover). The ratio of *unspliced* (pre-mRNA, containing introns) to *spliced*
(mature mRNA) for a given gene reflects the trajectory of its expression: high
unspliced relative to spliced means the gene is being turned on; low unspliced
relative to spliced means it is being turned off. RNA velocity uses this ratio to
infer a "velocity vector" for each cell — pointing in the direction it is
transcriptionally moving.

**Tools:**
- **scVelo** (Bergen et al., 2020): the standard implementation. Offers two
  models:
  - *Stochastic model*: assumes constant kinetics, fast. Good for a first pass.
  - *Dynamical model*: infers transcription, splicing, and degradation rates per
    gene. More accurate, especially for genes with heterogeneous kinetics.
    Computationally expensive but worth it for final analyses.
- **velociraptor** (R wrapper for scVelo, integrates with Bioconductor)
- **CellRank** (Lange et al.): extends RNA velocity with Markov chain models to
  compute transition probabilities and terminal state probabilities.

**Requirements:** You need both spliced and unspliced count matrices. These are
generated during alignment: STARsolo and alevin-fry support this natively; for
Cell Ranger output, use velocyto to re-quantify or generate unspliced counts from
the BAM.

**Biological interpretation:** Velocity arrows on UMAP indicate the direction of
transcriptional change. A progenitor population with arrows pointing toward
differentiated clusters supports the proposed differentiation trajectory. Velocity
arrows pointing in circles indicate unstable kinetics, overdetermined models, or
that the dataset violates velocity assumptions (e.g. no genuine differentiation
is occurring in a snapshot dataset from a stable tissue).

**When RNA velocity fails:**
- Tissues with very stable, non-differentiating cell types (e.g. fully mature
  neurons, terminally differentiated T cells in peripheral blood) have low
  unspliced/spliced ratio variation — not enough signal
- The model assumes a single kinetic regime per gene, which fails for genes with
  complex regulation
- Poor coverage of intronic reads (some library prep kits deplete unspliced RNA)

---

## 6. scATAC-seq — Chromatin Accessibility per Cell

**What it measures:** ATAC-seq (Assay for Transposase-Accessible Chromatin)
uses the hyperactive transposase Tn5 to insert sequencing adapters preferentially
into nucleosome-free (open) chromatin. In scATAC-seq, this is done at single-cell
resolution — you learn which genomic regions are accessible in each cell. Open
chromatin correlates with active regulatory elements (promoters, enhancers).

**scATAC-seq vs scRNA-seq:** scRNA-seq measures what genes are active (output);
scATAC-seq measures what regulatory elements are accessible (the regulatory
state that drives output). scATAC-seq gives you the upstream regulatory landscape;
scRNA-seq gives you the functional consequence. They are complementary.

**Fragment size distribution as QC:** Because Tn5 inserts in nucleosome-free
regions, fragments come in discrete size classes: sub-nucleosomal (~100 bp,
accessible region between nucleosomes), mono-nucleosomal (~200 bp, one
nucleosome's footprint), di-nucleosomal (~400 bp), and so on. A high-quality
scATAC-seq experiment shows clear banding in the fragment size histogram. A flat
or smeared distribution indicates poor transposase efficiency or degraded chromatin.
This is one of the first QC checks to perform.

**Per-cell QC metrics:**
- **TSS enrichment score:** fragments should be enriched at transcription start
  sites (open chromatin) relative to flanking regions. Score > 4 is generally
  acceptable; > 8 is high quality. Below 4 indicates poor signal-to-noise.
- **Number of fragments per cell:** cells with < 1,000 fragments are poorly
  covered. Cells with > 100,000 fragments are likely doublets.
- **Fraction of reads in peaks (FRiP):** same concept as ChIP-seq FRiP. > 15–20%
  is acceptable for scATAC-seq.

**Peak calling:** You cannot call peaks per cell (too few fragments). Peaks are
called on pseudo-bulk profiles: aggregate fragments from all cells of the same
cell type, call peaks with MACS2, then count per-cell fragments in those peaks.

**Linking peaks to genes:**
- Nearest gene: assign each peak to the nearest gene TSS. Simple but inaccurate
  (many enhancers act on non-nearest genes).
- Activity-by-contact model (Fulco et al.): incorporates Hi-C or Hi-ChIP contact
  frequency to weight peak-gene links by 3D proximity. More accurate but requires
  3D genomics data.
- Correlation-based: in multiome data, directly correlate peak accessibility with
  gene expression across cells.

**Tools:**
- **ArchR** (Granja et al.): full R-based pipeline for scATAC-seq. Handles
  everything from fragment files to clustering, motif enrichment, and peak-gene
  links. Highly optimised for large datasets.
- **Signac** (Stuart et al.): extends Seurat for scATAC-seq. Best when you want
  a unified scRNA-seq + scATAC-seq workflow in Seurat.

**10x Multiome:** The 10x Multiome kit measures RNA and ATAC simultaneously in
the same cell, using a single nucleus (not whole cell — nuclei are required to
keep chromatin intact during processing). This is the most powerful approach for
linking regulatory state to gene expression. Limitations: nuclear RNA only (no
cytoplasmic mRNA, so some transcripts are lower abundance); requires fresh tissue
or very careful nuclei isolation.

---

## 7. Multi-Sample and Clinical scRNA-seq

Single-cell experiments increasingly involve multiple donors, time points, or
experimental conditions. The analysis of these datasets requires additional
considerations beyond single-sample analysis.

### Sample Demultiplexing

When multiple samples are pooled into one 10x lane (multiplexing to reduce cost
and batch effects), you need to assign each cell to its sample of origin.

- **Genetic demultiplexing (vireo, demuxlet):** Uses natural genetic variants
  (SNPs) observed in the scRNA-seq reads. Each donor has a unique genotype; by
  matching the SNP profile of each cell's reads against a reference genotype
  (demuxlet) or by inferring genotypes from the data itself (vireo/cellSNP-lite),
  cells are assigned to donors. This also identifies doublets (cells with mixed
  genotypes). Works without prior genotyping if you have enough cells per donor.

- **Hashtag oligos (HTOs, Cell Hashing):** Label each sample with a unique
  antibody-oligonucleotide conjugate before pooling. The oligo barcode is
  sequenced alongside the transcriptome. Demultiplex with HTODemux (Seurat) or
  GMM-demux. Simpler to implement but requires antibody labelling.

### Pseudobulk Differential Expression

**The critical principle:** Cells from the same donor are not independent
observations. If donor A has 1,000 CD4 T cells and donor B has 800 CD4 T cells,
these are not 1,800 independent data points — they are two donors. Running
standard differential expression (Wilcoxon, MAST) on individual cells treats
them as independent and massively inflates the effective sample size, producing
false positives at catastrophically high rates.

**Pseudobulk analysis:**
1. For each cell type, aggregate counts across all cells of that type per sample
   (sum of raw counts per donor per cell type)
2. This produces a "pseudobulk" count matrix: samples × genes, one matrix per
   cell type
3. Apply bulk RNA-seq differential expression methods (DESeq2, edgeR, limma-voom)
   to this matrix

This is statistically correct. It treats donors as the unit of replication.
The cost: you need at least 3–4 samples per condition (ideally 6+) and sufficient
cells per cell type per sample to construct a reliable pseudobulk profile (at
least 20–50 cells per cell type per sample is recommended).

Reference: Squair et al. (2021) Nature Communications showed that single-cell
DE methods have inflated false positive rates up to 70% in multi-sample settings.
Pseudobulk methods have proper type I error control.

### Cell Type Composition Analysis

When asking "does disease change how many of each cell type are present?", the
appropriate analysis is compositional, not differential expression.

- **Milo** (Dann et al., 2021): tests for differential abundance in graph
  neighbourhoods rather than discrete clusters. More sensitive than cluster-based
  methods, avoids the hard cluster assignment problem.
- **DA-seq** (Zhao et al.): logistic regression-based approach that identifies
  cell state regions enriched in one condition.
- **propeller** (Phipson et al., 2022): tests cell type proportions using a linear
  model with logit-transformed proportions. Simple, well-calibrated, integrates
  with limma.

All of these methods require multiple samples per condition. A single case and
a single control give you no statistical power for composition analysis.

---

## Quick Benchmarks: Single-Cell

| Metric | Expected range | Red flag |
|---|---|---|
| Genes per cell (10x Chromium, typical tissue) | 1,000–4,000 | < 200 (dead cell) or > 8,000 (doublet) |
| UMIs per cell (10x Chromium) | 2,000–20,000 | < 500 or > 50,000 |
| % mitochondrial reads | 2–20% (tissue-dependent) | > 30% (dying cells) |
| Doublet rate (5,000 cells targeted, 10x) | ~4% | > 15% (too many cells loaded) |
| TSS enrichment (scATAC-seq) | > 4 (acceptable), > 8 (good) | < 2 (poor quality) |
| FRiP (scATAC-seq) | > 15% | < 5% |
| Fragment size banding visible (scATAC-seq) | Yes | No (chromatin degraded) |
| Cells per cluster (minimum meaningful) | > 50 | < 20 (statistical inference unreliable) |
| Samples per condition for pseudobulk DE | ≥ 4 | < 3 (underpowered) |
| Ambient RNA contamination (SoupX estimate) | 2–15% | > 30% (poor dissociation) |

---

## Teaching Approach for Different Learners

### For wet lab biologists

The most powerful bridge: **single-cell is like FACS, but instead of reading
surface markers, you read the whole transcriptome.**

In FACS, you gate on CD3 and CD8 to identify CD8 T cells. In scRNA-seq, you
cluster cells and look for high expression of CD3D, CD8A — you're doing the
same gating, computationally, on 20,000 dimensions instead of 2–8.

The sorting analogy extends further:
- FACS scatter plots = UMAP
- Gating = clustering (but without having to decide in advance what to look for)
- Surface marker panel = marker gene set
- Doublet in FACS (two cells passing through together) = doublet in scRNA-seq

What this framing unlocks: wet lab biologists immediately understand why doublets
are bad, why marker specificity matters for annotation, and why you need a large
enough sample to find rare populations (power is just sample size).

Emphasise that their biological intuition is the most important input to the
analysis. The algorithm finds the clusters; the biologist decides what they are.

**Controls:** They understand controls. Translate: empty droplets (no-cell
wells) are your blank control. If you sequence only the ambient RNA in your
solution, what would you see? That's what you're correcting with SoupX.

### For CS / data scientists

The matrix is the entry point: a cells × genes sparse count matrix, typically
10,000 rows × 30,000 columns, with ~95% zeros. This is the rawest representation
of single-cell data, and it lives in memory as a sparse matrix (CSR or CSC format).
The sparsity is not missing data — it is a fundamental property of single-cell
measurement. A zero may mean the gene is not expressed, or it may mean the mRNA
was not captured (dropout). This ambiguity is irreducible at the single-cell level.

The algorithm stack maps cleanly to familiar operations:
- Normalisation = feature scaling (but per-sample, not per-feature)
- HVG selection = variance-based feature selection
- PCA = standard dimensionality reduction; the compressed representation used
  for all downstream work
- KNN graph = the central data structure; all clustering and trajectory work
  operates on this graph
- Leiden/Louvain = graph community detection (same algorithms as network science)
- UMAP = non-linear dimensionality reduction for visualisation only

The central warning for data-first learners: **the pipeline can run without errors
and produce a beautiful UMAP that is biologically meaningless.** A UMAP of random
data also has clusters. The algorithm does not know biology. Every computational
output requires biological validation that requires domain knowledge. Do not
interpret clusters without knowing what marker genes look like in those clusters.

### For PhD students

Develop two reflexes before every analysis:

**Reflex 1 — Before looking at clusters:** Three questions to answer in this
order:
1. Did I remove doublets? (If no: some of your clusters are artefacts)
2. Did I correct for batch? (If no: some of your clusters are technical)
3. Did I check ambient RNA contamination? (If no: some of your marker genes are
   background)

These are not optional. Skipping any of them means you cannot trust your
biological conclusions.

**Reflex 2 — Before claiming a novel cell type:** What are the top 5 marker genes
of this cluster? Are they expressed in the cells you think they are by other
methods (immunofluorescence, flow cytometry, in situ hybridisation)? Is this
cluster absent when you remove doublets? Does it appear in independent datasets
from the same tissue? A novel cell type requires orthogonal validation.

The biggest mistake PhD students make: trusting a UMAP. The UMAP is a
communication tool. It compresses 20,000 dimensions into 2 for human eyes. The
position of a cluster on UMAP is not its identity, its distance from other
clusters is not its transcriptional similarity, and a cluster that looks separate
on UMAP may merge completely at the PCA level. Always go back to the data.

---

## Key Tools Summary

| Task | Primary tool | Alternative |
|---|---|---|
| Alignment (droplet) | Cell Ranger | STARsolo, alevin-fry |
| Alignment (plate) | STAR + featureCounts | kallisto + bustools |
| Cell barcode calling | Cell Ranger knee / EmptyDrops | DropUtils |
| QC and analysis (R) | Seurat | Bioconductor (scran, scater) |
| QC and analysis (Python) | Scanpy (AnnData) | — |
| Doublet detection | scDblFinder | Scrublet, DoubletFinder |
| Ambient RNA | SoupX | DecontX |
| Batch correction | Harmony | scVI, Seurat RPCA, BBKNN |
| Trajectory (pseudotime) | Monocle 3 | PAGA + DPT (Scanpy) |
| RNA velocity | scVelo | CellRank |
| scATAC-seq | ArchR | Signac |
| Cell type annotation | Azimuth, SingleR | Manual marker genes |
| Pseudobulk DE | DESeq2, edgeR | limma-voom |
| Composition analysis | Milo | propeller, DA-seq |
| Genetic demultiplexing | vireo + cellSNP-lite | demuxlet |
