# Changelog

All notable changes to bench2bash-skills are documented here.

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
