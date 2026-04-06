# Changelog

All notable changes to bench2bash-skills are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

---

## [1.1.0] — 2026-04-06

### Added
- `bioinformatics-tutor/references/single-cell.md`: Full teaching reference for
  scRNA-seq and scATAC-seq — UMI barcoding, QC filtering, normalisation, PCA/UMAP,
  clustering, doublet and ambient RNA detection, batch correction, trajectory
  analysis (Monocle3, scVelo), pseudobulk DE, and scATAC-seq workflow.
- `bioinformatics-fundamentals/references/workflow-reproducibility.md`: Conda/Mamba
  environment management, Docker/Singularity/BioContainers, Snakemake (rules,
  wildcards, cluster execution, profiles), Nextflow/nf-core pipelines, version
  control for bioinformatics, FAIR data practices, SLURM/HPC patterns, and
  reproducible notebooks.
- Oxford Nanopore (ONT) section in `reference.md`: Dorado basecalling (hac/sup/
  duplex/methylation modes), QC metrics, minimap2 alignment for standard and
  ultra-long reads, error profile guidance.
- Structural variant calling in `reference.md`: Manta, DELLY, SMOOVE (short-read);
  Sniffles2 (long-read, single and multi-sample); SURVIVOR multi-caller merge.
- Variant annotation in `reference.md`: VEP (all flags, full consequence severity
  ranking), SnpEff, gnomAD allele frequency filtering table, pathogenicity scores
  (CADD, SIFT, PolyPhen-2, GERP++, PhyloP).
- Hi-C scaffolding tools in `reference.md`: YAHS, SALSA2, 3D-DNA comparison table,
  Juicebox curation workflow, contact map misassembly signatures.
- DNA methylation in `reference.md`: Bismark WGBS pipeline, ONT modkit direct
  methylation calling, MethylKit/BSmooth DMR analysis, conversion efficiency QC.
- GitHub community health files: `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`,
  `SECURITY.md`, issue templates (skill-gap, bug-report, feature-request),
  PR template, GitHub Actions (lint + release workflows), `.gitignore`,
  `.markdownlint.yml`.

### Changed
- `concepts.md` expanded from 266 to 453 lines: deeper dives added to variant
  calling (SVs, CNVs, phasing, VQSR, annotation pipeline), RNA-seq (library
  prep, pseudoalignment decision tree, batch correction), ChIP-seq (input
  controls, IDR, motif analysis, blacklist regions), metagenomics (OTU vs ASV,
  compositionality, HUMAnN3), pangenomics (ANI species boundary, recombination
  detection), metabolic modelling (medium constraints, gap-fill pitfalls). New
  section 9 on single-cell sequencing. Expanded benchmarks table.
- Both `SKILL.md` trigger descriptions updated to cover new domains (ONT,
  structural variants, methylation, workflow reproducibility, single-cell).

### Fixed
- `common-issues.md`: clarified samtools off-by-one example — position 0 is
  invalid in samtools (1-based tool), not a "0-based" interpretation.
- `reference.md` Hi-C BWA-MEM2 command: added explicit breakdown of `-F 2316`
  flag composition and explanation of why proper-pair (`-f 2`) is intentionally
  absent for Hi-C alignment.
- `bioinformatics-tutor/SKILL.md`: added body reference to `learning-paths.md`
  in the "Detect the Learner's Background" section (was listed in frontmatter
  only).

---

## [1.0.0] — 2026-04-04

### Added
- `bioinformatics-tutor` skill: Socratic teaching skill for developing biological
  intuition. Covers genome assembly, RNA-seq, variant calling, ChIP-seq,
  metagenomics, bacterial pangenomics, and metabolic modelling.
- `bioinformatics-fundamentals` skill: Technical reference for SAM/BAM internals,
  AGP format, paired-end filtering patterns, sequencing technology specs, and
  assembly QC metrics.
- `bioinformatics-tutor/references/concepts.md`: Biological story and validation
  benchmarks for all major bioinformatics domains.
- `bioinformatics-tutor/references/learning-paths.md`: Structured learning
  progressions for wet lab biologists, CS/data scientists, and early PhD students.
- `bioinformatics-fundamentals/references/reference.md`: Complete SAM/BAM mandatory
  fields, all 12 FLAG bits, optional tags, CIGAR operations, AGP column specification,
  FASTQ Phred encoding tables, tool command reference (samtools, bamtools, minimap2,
  BWA-MEM2), sequencing technology specs, assembly QC metrics, and coordinate system
  conversion guide.
- `bioinformatics-fundamentals/references/common-issues.md`: Full troubleshooting
  guide covering empty output after region filtering, paired-end count mismatches,
  low mapping rate decision tree, Hi-C-specific issues (dangling ends, cis/trans
  ratio), HiFi-specific issues (CCS vs subread BAM, MAPQ=0 interpretation, kinetics
  tags), AGP processing errors, format conversion issues, and diagnostic command
  reference.
- `bioinformatics-fundamentals/references/genomeark-data-access.md`: GenomeArk
  AWS S3 directory structure for all VGP phases (v1, v1.6, v2, v3), QC data
  locations (GenomeScope2, BUSCO, Merqury), AWS CLI access patterns, path
  normalisation strategies, and assembly date extraction from filenames.
- `bioinformatics-fundamentals/references/genomic-analysis-patterns.md`: Karyotype
  data curation workflow, 2n/n/NF notation guide, haploid vs diploid assembly
  interpretation, phylogenetic tree species name mapping, NCBI synonym resolution,
  telomere BED analysis with tidk, T2T completeness assessment, and NCBI Datasets
  CLI patterns.
- Packaged `.skill` files in `releases/` for one-click installation.

### Changed
- Repo structure flattened: skill directories are now at the repository root
  (previously nested inside a `bench2bash-skills/` subdirectory).
- `bioinformatics-fundamentals` Related Skills section updated to reference only
  the co-packaged `bioinformatics-tutor` skill; removed references to skills that
  are not part of this repository.
