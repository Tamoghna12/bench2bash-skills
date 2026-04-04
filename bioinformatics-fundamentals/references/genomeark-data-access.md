# GenomeArk Data Access

Reference for accessing VGP (Vertebrate Genomes Project) assembly and raw data
from the GenomeArk AWS S3 repository. All data is publicly readable without
credentials using `--no-sign-request`.

---

## Table of Contents

1. [Overview](#overview)
2. [AWS CLI Setup](#aws-cli-setup)
3. [S3 Directory Structure](#s3-directory-structure)
4. [Directory Structure Evolution](#directory-structure-evolution)
5. [QC Data Locations](#qc-data-locations)
6. [Fetching Strategies](#fetching-strategies)
7. [Path Normalisation](#path-normalisation)
8. [Assembly Date Extraction](#assembly-date-extraction)

---

## Overview

**Bucket:** `s3://genomeark`
**URL base:** `https://genomeark.s3.amazonaws.com/`
**Access:** Public, no authentication needed. Use `--no-sign-request` with AWS CLI.

**Cost note:** Downloading from S3 to AWS infrastructure in the same region (us-east-1)
is free. Downloads to external networks incur AWS egress charges (~$0.09/GB).
For large downloads, use an EC2 instance in `us-east-1`.

**Species naming convention:** `<CommonName>_<GenusSpecies>` where genus is
capitalised and species is lowercase.
- Example: `Lynx_canadensis` (Canada lynx)
- Example: `Homo_sapiens` (human, rarely on GenomeArk)

**ToLID (Tree of Life ID):** Species identifier used in filenames.
- Format: `<prefix><letter><number>` (e.g. `mLynCan4`, `bTaeGut1`)
- Prefix encodes taxonomic group: `m` = mammal, `b` = bird, `f` = fish,
  `r` = reptile, `i` = insect, etc.
- Number = individual number for that species

---

## AWS CLI Setup

```bash
# Install AWS CLI v2
# Linux:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install

# macOS:
brew install awscli

# Test access (no credentials needed for public data)
aws s3 ls --no-sign-request s3://genomeark/species/ | head -10
```

No `aws configure` needed for public access — just always append `--no-sign-request`.

---

## S3 Directory Structure

```
s3://genomeark/species/
  <CommonName_GenusSpecies>/          e.g. Lynx_canadensis/
    <ToLID>/                          e.g. mLynCan4/
      genomic_data/
        pacbio_hifi/
          <ToLID>.hifi_reads.<date>.<ext>   # CCS BAM or FASTQ
        hic/
          <ToLID>.hic.<date>.<ext>          # Hi-C FASTQ pairs
        illumina/
          <ToLID>.illumina.<date>.<ext>     # Illumina FASTQ
        pacbio_clr/                         # Legacy; subreads
        bionano/                            # Optical maps
        phase10x/                           # 10x Genomics linked reads (v1 era)
      assembly_curated/
        <ToLID>.<version>/
          <ToLID>.<version>.pri.cur.<YYYYMMDD>.fasta.gz    # Primary haplotype
          <ToLID>.<version>.alt.cur.<YYYYMMDD>.fasta.gz    # Alternate haplotype
          <ToLID>.<version>.pri.cur.<YYYYMMDD>.agp          # AGP for primary
          <ToLID>.<version>.alt.cur.<YYYYMMDD>.agp          # AGP for alternate
      assembly_pipeline/
        <pipeline_version>/                  # Raw pipeline outputs (less curated)
      transcriptomic_data/
        illumina/
          <ToLID>.rna.<tissue>.<date>.<ext>
```

**Example paths for Canada lynx (mLynCan4):**

```
s3://genomeark/species/Lynx_canadensis/mLynCan4/genomic_data/pacbio_hifi/
s3://genomeark/species/Lynx_canadensis/mLynCan4/assembly_curated/mLynCan4.pri.v3/
s3://genomeark/species/Lynx_canadensis/mLynCan4/assembly_curated/mLynCan4.pri.v3/mLynCan4.pri.v3.pri.cur.20230501.fasta.gz
```

---

## Directory Structure Evolution

The VGP directory layout has changed across project phases. Older species
use different path patterns:

### VGP v1 (2018–2020)

```
s3://genomeark/species/<Name>/<ToLID>/
  genomic_data/10x/                # 10x Genomics linked reads
  genomic_data/pacbio/             # CLR subreads (not CCS)
  genomic_data/bionano/
  genomic_data/hi-c/               # Note: hyphen, not underscore
  assembly_v1.6/                   # Version naming style
    <ToLID>_v1.6.fasta.gz
    <ToLID>_v1.6.agp
```

### VGP v1.6 (2020–2021)

Transition period. Some species have both `assembly_v1.6/` and
`assembly_curated/` directories.

### VGP v2 (2021–2023)

```
genomic_data/pacbio_hifi/          # CCS reads (replaces clr)
assembly_curated/<ToLID>.pri.v2/   # New versioning scheme
```

### VGP v3 / Earth BioGenome Project era (2023–present)

```
genomic_data/pacbio_hifi/
genomic_data/hic/                  # Arima Hi-C
assembly_curated/<ToLID>.pri.v3/
  *.pri.cur.YYYYMMDD.fasta.gz      # Date-stamped
  *.pri.cur.YYYYMMDD.agp
```

**Parsing old paths:** When writing scripts that handle multiple versions,
check for the presence of directory names rather than assuming a fixed path:

```bash
# List all assembly versions for a species
aws s3 ls --no-sign-request "s3://genomeark/species/Lynx_canadensis/mLynCan4/" \
  | grep "assembly_"
```

---

## QC Data Locations

QC outputs are stored within the assembly directory, typically under a `qc/`
or `evaluation/` subdirectory.

### GenomeScope2

```
assembly_curated/<ToLID>.<version>/qc/
  <ToLID>.<version>.genomescope.model.txt     # Genome size, heterozygosity
  <ToLID>.<version>.genomescope.summary.txt
  <ToLID>.<version>.genomescope.linear_plot.png
```

```bash
aws s3 ls --no-sign-request \
  "s3://genomeark/species/Lynx_canadensis/mLynCan4/assembly_curated/mLynCan4.pri.v3/qc/"
```

### BUSCO

```
assembly_curated/<ToLID>.<version>/qc/busco/
  short_summary.specific.<lineage>.<ToLID>.txt
  full_table.tsv
```

```bash
aws s3 cp --no-sign-request \
  "s3://genomeark/species/.../qc/busco/short_summary.specific.mammalia_odb10.mLynCan4.txt" \
  ./busco_summary.txt
```

### Merqury

```
assembly_curated/<ToLID>.<version>/qc/merqury/
  <ToLID>.<version>.qv              # Per-sequence QV scores
  <ToLID>.<version>.completeness.stats
  <ToLID>.<version>.spectra-cn.fl.png
```

### Assembly Stats

```
assembly_curated/<ToLID>.<version>/qc/
  <ToLID>.<version>.assembly_stats.txt    # seqkit stats or assembly-stats output
```

---

## Fetching Strategies

### List Contents

```bash
# List species directory
aws s3 ls --no-sign-request s3://genomeark/species/ | head -20

# List all files for a ToLID
aws s3 ls --no-sign-request --recursive \
  s3://genomeark/species/Lynx_canadensis/mLynCan4/assembly_curated/ \
  | grep -v "/$"  # exclude directory entries

# Find all fasta.gz files for a species
aws s3 ls --no-sign-request --recursive \
  s3://genomeark/species/Lynx_canadensis/ \
  | grep "\.fasta\.gz$"
```

### Download Specific Files

```bash
# Download a single assembly FASTA
aws s3 cp --no-sign-request \
  s3://genomeark/species/Lynx_canadensis/mLynCan4/assembly_curated/mLynCan4.pri.v3/mLynCan4.pri.v3.pri.cur.20230501.fasta.gz \
  ./mLynCan4.pri.fasta.gz

# Download with progress indicator
aws s3 cp --no-sign-request --progress \
  s3://genomeark/species/Lynx_canadensis/mLynCan4/genomic_data/pacbio_hifi/ \
  ./hifi_reads/ --recursive
```

### Bulk Download with sync

```bash
# Sync an entire assembly directory (only downloads new/changed files)
aws s3 sync --no-sign-request \
  s3://genomeark/species/Lynx_canadensis/mLynCan4/assembly_curated/mLynCan4.pri.v3/ \
  ./mLynCan4_v3/

# Dry run first to see what would be downloaded
aws s3 sync --no-sign-request --dryrun \
  s3://genomeark/species/Lynx_canadensis/mLynCan4/ ./mLynCan4/
```

### Stream Without Downloading

```bash
# Stream a FASTA file and pipe directly to seqkit stats (no local copy)
aws s3 cp --no-sign-request \
  s3://genomeark/.../assembly.fasta.gz - | zcat | seqkit stats -

# Stream BAM index to check what's available
aws s3 cp --no-sign-request s3://genomeark/.../reads.bam.bai - > reads.bam.bai
```

---

## Path Normalisation

GenomeArk paths contain common inconsistencies. When building automated
pipelines, normalise aggressively:

### Species Name Capitalisation

The first word (common name) uses Title Case; the genus is capitalised; species
is lowercase:
- Correct: `Lynx_canadensis`
- Wrong: `lynx_canadensis`, `LYNX_CANADENSIS`

```python
def normalise_species_name(name):
    """Normalise to GenomeArk convention: First_second → First_second."""
    parts = name.replace('-', '_').split('_')
    if len(parts) >= 2:
        return '_'.join([parts[0].capitalize()] + [p.lower() for p in parts[1:]])
    return name
```

### ToLID Case

ToLIDs are always mixed case: `mLynCan4` not `mlyncan4` or `MLYNCAN4`.

```python
import re
TOLID_PATTERN = re.compile(r'^[a-z][A-Z][a-zA-Z]+\d+$')
```

### Trailing Slashes

`aws s3 ls` always uses trailing slashes for directories. Strip them when
building file paths:

```bash
species_path="s3://genomeark/species/Lynx_canadensis/mLynCan4/"
# Strip trailing slash for file operations
species_path="${species_path%/}"
```

### Filename Date Component

Assembly files include a date stamp in `YYYYMMDD` format:

```
mLynCan4.pri.v3.pri.cur.20230501.fasta.gz
                          ^^^^^^^^
```

When multiple dates exist (re-curations), always take the most recent:

```bash
# Find most recent assembly FASTA
aws s3 ls --no-sign-request "s3://genomeark/.../assembly_curated/mLynCan4.pri.v3/" \
  | grep "\.fasta\.gz$" \
  | awk '{print $NF}' \
  | sort -t. -k5 -V \
  | tail -1
```

---

## Assembly Date Extraction

Assembly filenames encode the curation date. This is important for
reproducibility and for tracking which version of the assembly was used.

**Filename pattern:**
```
<ToLID>.<version>.<haplotype>.cur.<YYYYMMDD>.fasta.gz
```

**Extract with regex:**

```python
import re
from datetime import datetime

def extract_assembly_date(filename):
    """Extract curation date from VGP assembly filename."""
    m = re.search(r'\.cur\.(\d{8})\.', filename)
    if m:
        return datetime.strptime(m.group(1), '%Y%m%d').date()
    return None

# Example
filename = 'mLynCan4.pri.v3.pri.cur.20230501.fasta.gz'
date = extract_assembly_date(filename)  # datetime.date(2023, 5, 1)
```

**Extract from S3 metadata (alternative):**

```bash
# Get S3 object metadata including last-modified date
aws s3api head-object --no-sign-request \
  --bucket genomeark \
  --key "species/Lynx_canadensis/mLynCan4/assembly_curated/mLynCan4.pri.v3/mLynCan4.pri.v3.pri.cur.20230501.fasta.gz" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['LastModified'])"
```

**Why date matters:**
- VGP assemblies are re-curated over time; the same version string (v3) can have
  multiple curation dates
- For paper methods: always record the exact filename including date stamp
- For reproducible pipelines: pin to a specific dated filename, not just the version
