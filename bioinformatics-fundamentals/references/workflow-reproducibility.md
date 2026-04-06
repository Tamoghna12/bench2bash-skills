# Workflow Management and Reproducibility

Computational reproducibility is a core requirement of rigorous bioinformatics. A result is only credible if someone else — or you, six months later — can regenerate it from the same inputs using the same software. This reference covers the tools and practices that make that possible.

---

## 1. Environment Management with Conda/Mamba

### Why Environments Matter

Every bioinformatics tool has dependencies: specific versions of libraries, compilers, or other tools. Without isolation, installing one tool can silently break another. Conda environments solve this by creating isolated namespaces where each project has its own dependency tree, independent of the system Python or other projects.

Version pinning is equally critical. `samtools 1.16` and `samtools 1.18` produce different output for edge cases. A methods section that says "we used samtools" is not reproducible. An environment file that pins `samtools=1.18` is.

### mamba vs conda

mamba is a drop-in replacement for conda that uses the `libsolv` dependency resolver (the same used by RPM/zypper). conda's default solver is slow and can take 10–30 minutes on complex environments. mamba typically resolves the same environment in under a minute.

Install mamba via miniforge (recommended over Anaconda for licensing reasons):

```bash
# Install miniforge (includes mamba)
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
bash Miniforge3-Linux-x86_64.sh
```

### Channels

Conda packages are distributed through channels:

| Channel | Contents | Notes |
|---|---|---|
| `conda-forge` | General scientific software | Community-maintained, most up-to-date |
| `bioconda` | Bioinformatics tools | Requires conda-forge as dependency channel |
| `defaults` | Anaconda-curated packages | Slower updates; commercial license for large orgs |

**Channel priority order:** `conda-forge` > `bioconda` > `defaults`

This ordering matters. If the same package exists in multiple channels, conda uses the highest-priority channel. Without this order, you risk getting an outdated `defaults` build that conflicts with bioconda tools.

Configure globally:

```bash
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
```

### Creating Environments

Always specify versions when creating environments for analysis work. Omitting versions produces non-reproducible results — the environment will install the latest available package at creation time, which will differ between users and time points.

```bash
# Create a named environment with pinned versions
mamba create -n rnaseq \
  -c conda-forge -c bioconda \
  star=2.7.11a \
  salmon=1.10.0 \
  samtools=1.18 \
  fastp=0.23.4 \
  multiqc=1.19

# Activate
conda activate rnaseq

# Verify key versions
samtools --version | head -1
STAR --version
salmon --version
```

### Exporting and Recreating Environments

`conda env export` captures the full resolved environment including all transitive dependencies:

```bash
# Export (with --no-builds omits platform-specific build strings, more portable)
conda env export --no-builds > environment.yml

# Recreate on another system
mamba env create -f environment.yml
```

A minimal `environment.yml` that specifies only direct dependencies (more maintainable):

```yaml
name: rnaseq
channels:
  - conda-forge
  - bioconda
  - defaults
dependencies:
  - star=2.7.11a
  - salmon=1.10.0
  - samtools=1.18
  - fastp=0.23.4
  - multiqc=1.19
  - python=3.11
```

### When environment.yml Is Not Enough: conda-lock

`environment.yml` is not fully reproducible across platforms because:
- Build strings differ between Linux and macOS
- Dependency resolution can produce different transitive versions over time
- `--no-builds` improves portability but sacrifices exact reproducibility

`conda-lock` generates a platform-specific lock file that records every package with its exact version, build string, and URL:

```bash
pip install conda-lock

# Generate lock files for multiple platforms
conda-lock -f environment.yml -p linux-64 -p osx-arm64

# Creates: conda-lock.yml (multi-platform) or environment-linux-64.lock
# Recreate exactly
conda-lock install --name rnaseq conda-lock.yml
```

Use `conda-lock` for any environment that must be exactly reproducible — publication analyses, shared pipelines, Docker image builds.

---

## 2. Containers (Docker and Singularity/Apptainer)

### Why Containers

Even with conda, reproducing an environment requires conda to be installed. Containers package the entire software stack — OS libraries, system dependencies, and application software — into a single portable image. The container runs identically on any host that has the container runtime installed.

This solves the "works on my machine" problem for complex pipelines with system-level dependencies (e.g., tools that link against specific glibc versions, Java runtimes, or GPU libraries).

### Docker vs Singularity/Apptainer

| Feature | Docker | Singularity/Apptainer |
|---|---|---|
| Requires root to run | Yes (daemon runs as root) | No (runs as invoking user) |
| HPC cluster support | Rarely allowed | Designed for HPC |
| Image format | Layers (OCI) | Single `.sif` file |
| Pull Docker images | Native | Yes (`singularity pull docker://`) |
| Bind mounts | `-v host:container` | `--bind host:container` |

**Apptainer** is the successor to Singularity after the project moved to the Linux Foundation. The CLI is largely compatible; most HPC systems have one or the other.

### BioContainers

BioContainers provides pre-built, versioned containers for every package in bioconda, hosted at `quay.io/biocontainers`. The image tag encodes the exact package version and build string, ensuring the container matches a specific conda build exactly.

```bash
# Singularity — pull once, run many times (recommended for HPC)
singularity pull docker://quay.io/biocontainers/samtools:1.18--h50ea8bc_1
singularity exec samtools_1.18--h50ea8bc_1.sif samtools view -c input.bam

# Bind a directory (required to access host files)
singularity exec --bind /data:/data samtools_1.18--h50ea8bc_1.sif \
  samtools sort -@ 8 -o /data/sorted.bam /data/input.bam

# Docker — better for local development and cloud
docker run --rm \
  -v $(pwd):/data \
  quay.io/biocontainers/samtools:1.18--h50ea8bc_1 \
  samtools view -c /data/input.bam
```

Find the exact tag for a tool: `https://quay.io/repository/biocontainers/samtools?tab=tags`

### Writing a Dockerfile for a Custom Pipeline

```dockerfile
# Multi-stage build: builder stage installs dependencies, final stage copies binaries
FROM condaforge/miniforge3:23.11.0-0 AS builder

COPY environment.yml /tmp/environment.yml
RUN mamba env create -f /tmp/environment.yml && \
    conda clean --all --yes

# Final stage: smaller image
FROM condaforge/miniforge3:23.11.0-0

COPY --from=builder /opt/conda/envs/rnaseq /opt/conda/envs/rnaseq

ENV PATH=/opt/conda/envs/rnaseq/bin:$PATH
ENV CONDA_DEFAULT_ENV=rnaseq

# Copy pipeline scripts
COPY scripts/ /pipeline/scripts/

WORKDIR /data
```

Multi-stage builds reduce final image size by not including the conda installer and build cache in the shipped image.

### Singularity Definition Files (.def)

For reproducible Singularity builds (as opposed to pulling from Docker Hub):

```singularity
Bootstrap: docker
From: quay.io/biocontainers/samtools:1.18--h50ea8bc_1

%post
    # Any additional setup
    apt-get update && apt-get install -y procps

%environment
    export PATH=/usr/local/bin:$PATH

%labels
    Author yourname@institution.edu
    Version 1.0
    SamtoolsVersion 1.18

%runscript
    exec samtools "$@"
```

Build: `sudo singularity build samtools_custom.sif samtools.def`

---

## 3. Snakemake — Workflow Management

### What Snakemake Does

Snakemake is a Python-based workflow manager. You define rules specifying what each step takes as input and produces as output. Snakemake automatically:
- Determines which rules need to run to produce the requested targets
- Parallelises independent steps
- Skips rules whose outputs already exist and are newer than inputs
- Resubmits failed jobs

The workflow definition (Snakefile) is version-controlled alongside analysis scripts, making the entire pipeline reproducible.

### Rule Anatomy

```python
# Snakefile

# Define samples at the top
SAMPLES = ["SRR001", "SRR002", "SRR003"]

rule all:
    input:
        expand("results/{sample}.bam", sample=SAMPLES)

rule align:
    input:
        reads="data/{sample}.fq.gz",
        ref="reference/genome.fa"
    output:
        bam="results/{sample}.bam",
        bai="results/{sample}.bam.bai"
    log:
        "logs/align/{sample}.log"
    threads: 8
    resources:
        mem_mb=32000
    shell:
        """
        minimap2 -a -x map-hifi -t {threads} {input.ref} {input.reads} \
          | samtools sort -@ {threads} -o {output.bam} 2>{log}
        samtools index {output.bam}
        """
```

Key concepts:
- **`rule all`**: The default target. Snakemake works backwards from this to determine what needs to run.
- **Wildcards**: `{sample}` in file paths is a wildcard. Snakemake matches it against actual files.
- **`expand()`**: Generates a list of file paths by filling in wildcard values: `expand("results/{sample}.bam", sample=SAMPLES)` produces `["results/SRR001.bam", "results/SRR002.bam", "results/SRR003.bam"]`.
- **`log:`**: Captures stderr/stdout to a file — essential for debugging.
- **`resources:`**: Declares memory/time requirements, used by cluster schedulers.

### Per-Rule Conda Environments

Snakemake can activate a different conda environment for each rule, enabling tools with conflicting dependencies to coexist in one pipeline:

```python
rule call_variants:
    input: "results/{sample}.bam"
    output: "results/{sample}.vcf"
    conda: "envs/gatk.yml"   # path to environment.yml for this rule
    shell:
        "gatk HaplotypeCaller -I {input} -O {output} -R reference/genome.fa"
```

Run with: `snakemake --use-conda`

### Per-Rule Containers

```python
rule align:
    input:
        reads="data/{sample}.fq.gz",
        ref="reference/genome.fa"
    output:
        bam="results/{sample}.bam"
    container:
        "docker://quay.io/biocontainers/minimap2:2.26--he4a0461_2"
    threads: 8
    shell:
        "minimap2 -a -x map-hifi -t {threads} {input.ref} {input.reads} "
        "| samtools sort -@ {threads} -o {output.bam}"
```

Run with: `snakemake --use-singularity`

### Dry Run and DAG Visualization

Always do a dry run before submitting a large job:

```bash
# Dry run: shows what would be executed without running anything
snakemake -n

# Dry run with reason for each rule execution
snakemake -n -r

# Visualize the directed acyclic graph (DAG) of jobs
snakemake --dag | dot -Tsvg > dag.svg

# Visualize the rule graph (collapsed, cleaner for large workflows)
snakemake --rulegraph | dot -Tpdf > rulegraph.pdf
```

### Cluster Execution (SLURM)

```bash
# Submit jobs to SLURM with per-job resource allocation
snakemake \
  --cluster "sbatch --mem={resources.mem_mb}M --cpus-per-task={threads} --time=4:00:00" \
  --jobs 100 \
  --latency-wait 60 \
  --rerun-incomplete

# With a SLURM profile (recommended for production)
snakemake --profile slurm --jobs 100
```

`--latency-wait 60` is important on networked filesystems: it tells Snakemake to wait up to 60 seconds for output files to appear after a job completes (NFS can lag).

### Snakemake Profiles

Profiles store cluster settings so you don't repeat them on every command. A minimal SLURM profile at `~/.config/snakemake/slurm/config.yaml`:

```yaml
cluster: "sbatch --mem={resources.mem_mb}M --cpus-per-task={threads} --time={resources.runtime}"
default-resources:
  - mem_mb=4000
  - threads=1
  - runtime="1:00:00"
jobs: 100
latency-wait: 60
use-conda: true
```

Then run: `snakemake --profile slurm`

---

## 4. Nextflow — Pipeline Framework

### When to Use Nextflow vs Snakemake

Both are mature workflow managers. Choose based on context:

| Criterion | Snakemake | Nextflow |
|---|---|---|
| Primary paradigm | File-based (output file = job identity) | Process-based (channels pass data) |
| Language | Python | Groovy/DSL2 |
| Cloud-native | Good with profiles | Excellent (AWS Batch, GCP, Azure) |
| HPC integration | Excellent | Good |
| Community pipelines | Some | nf-core (200+ pipelines) |
| Learning curve | Lower for Python users | Higher |

Use Nextflow when: deploying large production pipelines, using nf-core pipelines, or running primarily on cloud infrastructure.

Use Snakemake when: writing analysis workflows, working primarily on HPC, or when your team is more comfortable with Python.

### DSL2 Syntax Basics

```nextflow
// main.nf

// Define a process
process ALIGN {
    container 'quay.io/biocontainers/minimap2:2.26--he4a0461_2'

    cpus 8
    memory '32 GB'

    input:
        path reads
        path ref

    output:
        path "*.bam"

    script:
        """
        minimap2 -a -x map-hifi -t ${task.cpus} $ref $reads \
          | samtools sort -o ${reads.baseName}.bam
        """
}

// Define a workflow
workflow {
    reads_ch = Channel.fromPath(params.reads)
    ref_ch   = Channel.value(file(params.ref))

    ALIGN(reads_ch, ref_ch)
}
```

Run: `nextflow run main.nf --reads 'data/*.fq.gz' --ref reference/genome.fa -profile singularity`

### nf-core

nf-core is a community effort to build best-practice Nextflow pipelines for standard bioinformatics analyses. Each pipeline is:
- Containerised (Docker + Singularity)
- Validated against reference datasets
- Configurable via a standardised parameter interface
- Documented with usage guides and output descriptions

Key pipelines:

| Pipeline | Use case |
|---|---|
| `nf-core/rnaseq` | Bulk RNA-seq (STAR/Salmon, DESeq2-ready counts) |
| `nf-core/sarek` | Germline + somatic variant calling (GATK, Strelka2, Manta) |
| `nf-core/atacseq` | ATAC-seq peak calling and QC |
| `nf-core/hic` | Hi-C contact map generation |
| `nf-core/nanoseq` | Oxford Nanopore long-read processing |
| `nf-core/ampliseq` | 16S/ITS amplicon sequencing (QIIME2) |

Running an nf-core pipeline:

```bash
# Install nf-core tools
pip install nf-core

# List available pipelines
nf-core list

# Download a pipeline for offline use (HPC clusters often lack internet)
nf-core download rnaseq --singularity-cache-only

# Run nf-core/rnaseq
nextflow run nf-core/rnaseq \
  -profile singularity \
  --input samplesheet.csv \
  --genome GRCh38 \
  --outdir results/ \
  -r 3.14.0    # pin the pipeline version
```

The `samplesheet.csv` format for nf-core/rnaseq:

```
sample,fastq_1,fastq_2,strandedness
control_rep1,/data/ctrl1_R1.fq.gz,/data/ctrl1_R2.fq.gz,auto
control_rep2,/data/ctrl2_R1.fq.gz,/data/ctrl2_R2.fq.gz,auto
treatment_rep1,/data/trt1_R1.fq.gz,/data/trt1_R2.fq.gz,auto
```

### Executors

Configure in `nextflow.config`:

```groovy
profiles {
    slurm {
        process.executor = 'slurm'
        process.queue    = 'standard'
        process.clusterOptions = '--account=myproject'
    }
    awsbatch {
        process.executor = 'awsbatch'
        process.queue    = 'nextflow-queue'
        aws.region       = 'us-east-1'
        workDir          = 's3://my-bucket/nextflow-work'
    }
}
```

---

## 5. Version Control for Bioinformatics

### What to Version Control

**Do version control:**
- Analysis scripts (Python, R, shell)
- Snakefiles and Nextflow pipelines
- Configuration files (`config.yaml`, `nextflow.config`)
- Sample sheets and metadata
- Environment specifications (`environment.yml`, `conda-lock.yml`)
- Small reference files (BED files with gene lists, adapter sequences)

**Do not version control:**
- Raw sequencing data (FASTQ, BAM) — too large; use SRA/ENA
- Large reference genomes — use dedicated storage
- Pipeline outputs — these are reproducible from inputs + code
- Conda environments themselves — version the spec file, not the environment

### .gitignore for Bioinformatics

```gitignore
# Sequencing data
*.bam
*.bai
*.cram
*.fastq.gz
*.fq.gz
*.fastq
*.fq

# Reference genomes
*.fa
*.fasta
*.fa.gz
*.fasta.gz
*.fa.fai

# Large intermediate outputs
results/
data/
*.vcf.gz
*.bcf

# Pipeline working directories
.snakemake/
work/           # Nextflow work directory
.nextflow/
.nextflow.log*

# Jupyter notebooks (strip outputs with nbstripout instead)
# .ipynb_checkpoints/

# R
.Rhistory
.RData
*.Rproj.user/
```

### Tagging Releases for Publications

When you submit a manuscript, tag the exact commit used for analysis:

```bash
git tag -a v1.0-submission -m "Code used for initial submission to Nature Methods"
git push origin v1.0-submission
```

Archive the tagged release on Zenodo for a citable DOI: connect your GitHub repository to Zenodo at `zenodo.org/account/settings/github`.

### Git LFS for Reference Files

For reference files that must be versioned but are too large for standard git (e.g., small reference genomes, curated databases):

```bash
git lfs install
git lfs track "*.fa"
git lfs track "*.gtf"
git add .gitattributes
git add reference/
git commit -m "Add reference genome via LFS"
```

---

## 6. Recording Tool Versions

### Why This Is Non-Negotiable

A methods section that says "reads were aligned to GRCh38 using STAR" cannot be reproduced. A methods section that says "reads were aligned to GRCh38 (Ensembl release 110) using STAR 2.7.11a with parameters --outSAMtype BAM SortedByCoordinate --quantMode TranscriptomeSAM" can be.

### Command-Line Version Capture

Capture versions into a file at the start of every analysis:

```bash
# Create a versions record
VERSIONS_FILE="results/versions.txt"
mkdir -p results

{
  echo "=== Analysis versions $(date) ==="
  echo ""
  samtools --version | head -1
  minimap2 --version
  STAR --version 2>&1 | head -1
  salmon --version 2>&1
  fastp --version 2>&1 | head -1
  echo ""
  python --version
  pip list | grep -E "pandas|numpy|scipy|pysam|snakemake"
  echo ""
  R --version | head -1
  Rscript -e "sessionInfo()" 2>/dev/null
} > "$VERSIONS_FILE"
```

### Snakemake Benchmark Directive

The `benchmark:` directive records wall clock time, CPU time, and maximum RSS memory for each rule execution:

```python
rule align:
    input:
        reads="data/{sample}.fq.gz",
        ref="reference/genome.fa"
    output:
        bam="results/{sample}.bam"
    benchmark:
        "benchmarks/{sample}.align.txt"
    threads: 8
    shell:
        "minimap2 -a -t {threads} {input.ref} {input.reads} | samtools sort -o {output.bam}"
```

The benchmark file contains tab-separated columns: `s` (seconds), `h:m:s`, `max_rss` (MB), `max_vms`, `max_uss`, `max_pss`, `io_in`, `io_out`, `mean_load`, `cpu_time`.

### MultiQC

MultiQC aggregates QC metrics from dozens of tools into a single interactive HTML report. Supported tools include FastQC, fastp, STAR, HISAT2, Salmon, samtools flagstat, Picard, GATK, and many others.

```bash
# Run MultiQC on an entire results directory
multiqc results/ logs/ -o results/multiqc/ --filename multiqc_report.html

# Force overwrite if report already exists
multiqc results/ -o results/multiqc/ -f

# Include specific modules only
multiqc results/ --module fastqc --module star --module salmon -o results/multiqc/
```

The HTML report allows interactive exploration of per-sample QC metrics and highlights outlier samples. Archive this file with your results.

---

## 7. Data Management and FAIR Principles

### Raw Data Handling

Raw data is irreplaceable. Follow these rules:

1. **Never modify raw files.** Write-protect them immediately after transfer: `chmod 444 raw/*.fastq.gz`
2. **Generate checksums before and after transfer.** A corrupted FASTQ will produce subtly wrong results with no error message.
3. **Back up to at least two physically separate locations.** A compute cluster is not a backup.

```bash
# Generate checksums for all raw data
md5sum raw/*.fastq.gz > raw/checksums.md5

# Verify integrity (run after any data transfer)
cd raw && md5sum -c checksums.md5

# SHA256 for higher assurance (required by some repositories)
sha256sum raw/*.fastq.gz > raw/checksums.sha256
```

### FAIR Data Principles

| Principle | Meaning | Practice |
|---|---|---|
| Findable | Data has a persistent identifier | Submit to GEO/SRA with accession numbers; archive code on Zenodo |
| Accessible | Data can be retrieved by identifier | Use public repositories with open access where possible |
| Interoperable | Data uses standard formats | Use SAM/BAM, VCF, BED, GTF — not bespoke formats |
| Reusable | Data has clear provenance and license | Include metadata, methods, and a clear data license |

### Data Repositories

| Repository | Data type | Notes |
|---|---|---|
| GEO (NCBI) | RNA-seq, ChIP-seq, ATAC-seq, microarray | Required by most journals for functional genomics |
| SRA (NCBI) | Raw sequencing reads (FASTQ) | Submit raw reads; GEO submission auto-creates SRA entries |
| ENA (EMBL-EBI) | Raw sequencing reads | European equivalent to SRA; data is mirrored |
| dbSNP/dbVar | Variants | For germline variants and structural variants |
| Zenodo | Any files up to 50 GB | Code, processed data, supplementary files; GitHub integration for DOI |
| Figshare | Any files | Alternative to Zenodo |

### SRA Submission Workflow

```bash
# Install SRA toolkit
conda install -c bioconda sra-tools

# Validate FASTQ before submission
fastq-dump --stdout SRR000001 | head -8   # test download

# For submission: use NCBI's web submission portal or SRA toolkit
# Compress FASTQs before upload
pigz -p 8 sample.fastq  # produces sample.fastq.gz

# Verify compressed integrity
md5sum sample.fastq.gz
```

Submit via the NCBI Submission Portal: `submit.ncbi.nlm.nih.gov`. Always submit raw reads, not adapter-trimmed or filtered reads — reviewers and re-analysts need the originals.

### Metadata Standards

Submitting metadata in a recognised standard format makes your data reusable:

- **MIMARKS**: Minimum Information about a MARKer gene Sequence (16S, ITS amplicon studies)
- **MIxS**: Minimum Information about any (X) Sequence — umbrella standard covering MIMS (metagenomes), MIGS (genomes), MIMARKS
- **ENA sample checklists**: Structured metadata forms for ENA submission covering host-associated, environmental, and clinical samples
- **ISA-Tab**: Investigation/Study/Assay tabular format for multi-omics studies

---

## 8. HPC Best Practices

### SLURM Basics

SLURM is the scheduler on most academic HPC clusters. Key commands:

```bash
# Submit a job
sbatch --mem=32G \
       --cpus-per-task=8 \
       --time=12:00:00 \
       --job-name=align \
       --output=logs/align_%j.out \
       --error=logs/align_%j.err \
       --partition=standard \
       script.sh

# Monitor your jobs
squeue -u $USER
squeue -u $USER --format="%.18i %.9P %.30j %.8u %.8T %.10M %.9l %.6D %R"

# Check completed job statistics (memory, runtime, exit code)
sacct -j 12345 --format=JobID,JobName,Elapsed,MaxRSS,ExitCode,State

# Cancel a job
scancel 12345

# Cancel all your jobs
scancel -u $USER
```

### Requesting Resources

Over-requesting wastes queue priority and delays other users. Under-requesting causes jobs to be killed when they exceed limits.

Rules of thumb:
- Request **~130% of expected runtime** (1.3x buffer)
- Request **~130% of expected peak memory**
- Check actual usage with `sacct` after pilot runs and adjust
- Use `--mem-per-cpu` instead of `--mem` when running multi-threaded tools (ensures memory scales with thread count)

```bash
# After a job completes, check actual usage
sacct -j 12345 --format=JobID,Elapsed,MaxRSS,ReqMem,CPUTime

# MaxRSS is in KB; convert to GB
# If MaxRSS = 18GB and you requested 32GB, your future request should be ~24GB
```

### Array Jobs

Array jobs submit many near-identical jobs with a single `sbatch` command. Each job gets a unique `$SLURM_ARRAY_TASK_ID`:

```bash
# Submit array of 50 jobs
sbatch --array=1-50 align_array.sh

# With a maximum of 10 running simultaneously (polite on shared clusters)
sbatch --array=1-50%10 align_array.sh
```

Array script pattern using a sample list:

```bash
#!/bin/bash
#SBATCH --mem=32G
#SBATCH --cpus-per-task=8
#SBATCH --time=4:00:00

# Read sample name from a list file (one sample per line)
SAMPLE=$(sed -n "${SLURM_ARRAY_TASK_ID}p" samples.txt)

conda activate rnaseq

STAR --runThreadN $SLURM_CPUS_PER_TASK \
     --genomeDir reference/star_index \
     --readFilesIn data/${SAMPLE}_R1.fq.gz data/${SAMPLE}_R2.fq.gz \
     --readFilesCommand zcat \
     --outSAMtype BAM SortedByCoordinate \
     --outFileNamePrefix results/${SAMPLE}/
```

### Storage Architecture

Most HPC systems have at least three storage tiers:

| Storage | Speed | Backed up | Use for |
|---|---|---|---|
| `$HOME` | Slow | Yes | Scripts, configs, environment specs |
| Project/Scratch (persistent) | Medium | Often yes | Raw data, final results |
| Local scratch (`/tmp` or `$TMPDIR`) | Fast (SSD/local disk) | No | Per-job temporary files |

**I/O bottleneck pattern**: Network-attached storage is the most common performance bottleneck in bioinformatics. For I/O-intensive steps:

```bash
#!/bin/bash
#SBATCH --mem=32G
#SBATCH --cpus-per-task=16

SAMPLE=$1
LOCAL_TMP=$TMPDIR/${SLURM_JOB_ID}
mkdir -p $LOCAL_TMP

# Copy input to local scratch
cp /project/data/${SAMPLE}.fq.gz $LOCAL_TMP/
cp /project/reference/genome.fa $LOCAL_TMP/

# Run on local disk
minimap2 -a -t $SLURM_CPUS_PER_TASK \
  $LOCAL_TMP/genome.fa \
  $LOCAL_TMP/${SAMPLE}.fq.gz \
  | samtools sort -@ $SLURM_CPUS_PER_TASK -T $LOCAL_TMP/sort_tmp \
  -o $LOCAL_TMP/${SAMPLE}.bam

samtools index $LOCAL_TMP/${SAMPLE}.bam

# Copy results back to project storage
cp $LOCAL_TMP/${SAMPLE}.bam /project/results/
cp $LOCAL_TMP/${SAMPLE}.bam.bai /project/results/

# Cleanup
rm -rf $LOCAL_TMP
```

---

## 9. Reproducible Analysis Notebooks

### Jupyter Notebooks

Jupyter notebooks are excellent for exploratory analysis and communicating results. Their main reproducibility pitfall: notebook cells can be run out of order, and the `.ipynb` file stores cell outputs (plots, tables) that can become stale relative to the code.

Best practices:
- Always use **Kernel > Restart and Run All** before sharing or committing
- Never commit notebooks with stale outputs — use `nbstripout`
- Number cells logically so out-of-order execution is obvious
- Use a fixed random seed for any stochastic analysis: `np.random.seed(42)`

```bash
# Install and configure nbstripout (strips outputs on every git commit)
pip install nbstripout
nbstripout --install  # installs as a git filter in the current repo

# Verify it's configured
cat .git/config | grep nbstripout
```

### Quarto and R Markdown

Quarto (the successor to R Markdown) renders documents that interleave code and output into HTML, PDF, or Word. Unlike Jupyter, the document is re-executed from scratch on each render, guaranteeing outputs match the code.

```bash
# Install Quarto: https://quarto.org/docs/get-started/
quarto render analysis.qmd --to html
quarto render analysis.qmd --to pdf

# R Markdown (older but still widely used)
Rscript -e "rmarkdown::render('analysis.Rmd', output_format='html_document')"
```

A minimal Quarto document header:

```yaml
---
title: "RNA-seq Differential Expression Analysis"
author: "Your Name"
date: today
format:
  html:
    toc: true
    code-fold: true
    embed-resources: true   # single self-contained HTML file
execute:
  echo: true
  warning: false
---
```

### Parameterized Notebooks

For running the same analysis on different datasets, parameterize the notebook rather than hardcoding values:

**Python (papermill):**

```bash
pip install papermill

# Run notebook with parameters
papermill analysis.ipynb results/SRR001_analysis.ipynb \
  -p sample_id SRR001 \
  -p min_coverage 10 \
  -p output_dir results/SRR001/
```

In the notebook, tag the parameters cell with `parameters` metadata, then all variables in that cell become overridable.

**R (knitr params):**

```yaml
---
title: "Sample QC Report"
params:
  sample_id: "SRR001"
  min_depth: 10
  output_dir: "results/SRR001"
---
```

```r
# In the notebook body, access params:
sample <- params$sample_id
min_depth <- params$min_depth
```

```bash
Rscript -e "rmarkdown::render('qc_report.Rmd',
  params=list(sample_id='SRR002', output_dir='results/SRR002'),
  output_file='results/SRR002/qc_report.html')"
```

---

## Summary Checklist

A reproducible bioinformatics analysis should satisfy:

- [ ] All software versions are pinned in an `environment.yml` or `conda-lock.yml`
- [ ] The pipeline is defined in Snakemake or Nextflow, not a series of manual commands
- [ ] All code is under version control with meaningful commit messages
- [ ] The exact code version used for publication is tagged
- [ ] Raw data integrity is verified with checksums
- [ ] Raw data is submitted to a public repository (SRA/ENA/GEO)
- [ ] A `versions.txt` file records the output of `--version` for all key tools
- [ ] Jupyter notebooks have outputs stripped (nbstripout installed)
- [ ] Results can be regenerated by a new user with: `mamba env create -f environment.yml && snakemake`
