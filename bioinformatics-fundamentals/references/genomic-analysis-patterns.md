# Genomic Analysis Patterns

Domain-specific analysis patterns for vertebrate genomics: karyotype curation,
chromosome count interpretation, phylogenetic tree species mapping, telomere
analysis, and NCBI data integration.

---

## Table of Contents

1. [Karyotype Data Curation](#karyotype-data-curation)
2. [Haploid vs Diploid Chromosome Counts](#haploid-vs-diploid-chromosome-counts)
3. [Phylogenetic Tree Species Mapping](#phylogenetic-tree-species-mapping)
4. [BED and Telomere Analysis](#bed-and-telomere-analysis)
5. [NCBI Data Integration](#ncbi-data-integration)

---

## Karyotype Data Curation

### Sources for Chromosome Count Data

Primary sources, in order of reliability:

1. **NCBI Genome** — Search by species name; "Chromosome count" field in genome
   metadata. Most reliable for model organisms.
2. **Animal Chromosome Count Database (ACCD)** — Specialist database for vertebrates;
   includes literature citations.
3. **Animal Genome Size Database (AGSD)** — C-value (genome size) data; also
   includes 2n counts.
4. **Primary karyotype literature** — Search PubMed for `"<species>" karyotype 2n`
   or `"<genus>" chromosomes`. Always prefer the original cytogenetic study.
5. **NCBI Taxonomy** — Has chromosome count annotations for some species in the
   `genome_records` field.

### Resolving Conflicting Counts

Conflicting counts in the literature are common and have multiple valid causes:

| Cause | Example | Resolution |
|-------|---------|------------|
| Intraspecific variation | Different populations have different 2n | Report the range; note geographic source |
| Subspecies variation | 2n differs between subspecies | Identify which subspecies the assembly represents |
| Sex chromosome systems | XY vs X0 gives different counts in males/females | Note sex of the individual; report sex-specific count |
| Robertsonian fusions | 2n varies with number of fusions present | Use the most common form for the population/subspecies |
| B chromosomes | Extra non-homologous chromosomes | Report 2n without Bs; note B presence separately |
| Counting errors | Historical misidentification of small chromosomes | Weight newer studies with larger sample sizes |

**Practical curation workflow:**

```python
# When building a species-to-karyotype lookup table:
karyotype_record = {
    "species": "Lynx canadensis",
    "2n": 38,
    "n": 19,
    "NF": 72,  # nombre fondamental (chromosome arm count)
    "source": "Wurster-Hill & Centerwall 1982 Cytogenet Cell Genet",
    "notes": "XY sex determination; no B chromosomes reported",
    "assembly_tolid": "mLynCan4",
    "assembly_sex": "female"
}
```

### Notation: 2n, n, and NF

| Symbol | Meaning | Assembly context |
|--------|---------|-----------------|
| 2n     | Diploid chromosome number (somatic cells) | Assembly scaffold count ≈ 2n (diploid) or n (haploid) |
| n      | Haploid number (gametes) = 2n/2 | Primary assembly targets n chromosomes + sex chromosome |
| NF     | Nombre fondamental = number of chromosome arms (biarmed + monoarmed) | Useful for comparing karyotypes across fusions |

---

## Haploid vs Diploid Chromosome Counts

### Key Definitions

**Diploid (2n):** The chromosome count in somatic cells — what you see in a
karyogram. A human has 2n = 46 (22 pairs of autosomes + XX or XY).

**Haploid (n):** The gamete chromosome count = 2n/2. Human n = 23.

**VGP assembly representation:** Primary assemblies target the **haploid**
representation — n chromosomes (+ the sex chromosome from the individual).
An alternative haplotype assembly represents the other haplotype.

### Expected Scaffold Count vs Assembly Type

| Assembly type | Expected scaffold count |
|---------------|------------------------|
| Contig-level  | Thousands–millions of contigs |
| Scaffold-level| Hundreds of scaffolds |
| Chromosome-level (haploid) | n + sex chromosome = n+1 for females (XX), n for males (XY) |
| Chromosome-level (diploid) | 2n chromosomes (phased, e.g. Hifiasm with Hi-C phasing) |

For a female mammal with 2n=38 and XX sex chromosomes:
- Haploid primary assembly: 19 autosomes + X = **20 chromosomes**
- Plus unplaced scaffolds and unloc scaffolds

### Common Confusing Cases

**XY males:** The Y chromosome is typically much smaller than the X and often
incompletely assembled. Expected scaffold counts for male assemblies are n+1
(19 autosomes + X + Y = 21 for 2n=38 mammals) but Y may be in many fragments.

**ZW sex determination (birds, some reptiles, fish):** Females are ZW,
males are ZZ. The W chromosome is heterochromatic and often fragmented.

**B chromosomes:** Supernumerary chromosomes; non-essential, variable in number
within individuals. Report primary 2n without Bs. Note: assembly scaffolds
that don't map to any chromosome may be B chromosomes.

**Robertsonian fusions:** Two acrocentric chromosomes fuse → 2n decreases by 1
with no change in gene content. Common in Mus musculus domesticus populations
where 2n can range from 22 to 40 depending on fusion state.

**Polyploids:** Plants are commonly polyploid (4n, 6n, etc.). Some fish (e.g.
salmonids) are ancient polyploids. For polyploids: expected primary assembly
haplotype = 1 subgenome = 2n / ploidy chromosomes.

### Interpreting VGP Assembly Scaffold Counts

A chromosome-level assembly for a mammal with n=21 should have:
- ~21 large scaffolds (autosomes + sex chromosome)
- Some unloc scaffolds (assigned to a chromosome but unplaced)
- Unplaced scaffolds (not assigned to any chromosome)

If the assembly has significantly more large scaffolds than n: check for
collapsed haplotypes, duplicate assemblies, or possible allelic scaffolds.

---

## Phylogenetic Tree Species Mapping

### The Naming Problem

Species names in phylogenetic trees often don't match assembly metadata exactly.
Common mismatches:
- Underscore vs space: `Homo_sapiens` vs `Homo sapiens`
- Abbreviated genus: `H. sapiens` vs `Homo sapiens`
- Subspecies: `Mus musculus domesticus` vs `Mus musculus`
- Synonyms: `Bos taurus` vs `Bos primigenius taurus`
- Common name used instead of scientific: `Canada lynx` vs `Lynx canadensis`

### Canonical Resolution Strategy

**Step 1 — Normalise to genus_species format:**

```python
import re

def normalise_species_name(name):
    """Return 'Genus species' canonical form."""
    # Remove underscores
    name = name.replace('_', ' ').strip()
    # Remove trailing subspecies
    parts = name.split()
    if len(parts) >= 2:
        return f"{parts[0].capitalize()} {parts[1].lower()}"
    return name
```

**Step 2 — Resolve synonyms via NCBI Taxonomy:**

```python
from Bio import Entrez
Entrez.email = "your@email.com"

def resolve_synonym(species_name):
    """Return current accepted name and taxid via NCBI Taxonomy."""
    handle = Entrez.esearch(db="taxonomy", term=f'"{species_name}"[All Names]')
    record = Entrez.read(handle)
    if record["IdList"]:
        taxid = record["IdList"][0]
        handle2 = Entrez.efetch(db="taxonomy", id=taxid, retmode="xml")
        tax_record = Entrez.read(handle2)
        return {
            "taxid": taxid,
            "scientific_name": tax_record[0]["ScientificName"],
            "synonyms": [s["Name"] for s in tax_record[0].get("OtherNames", {}).get("Synonym", [])]
        }
    return None
```

**Step 3 — Open Tree of Life for broad phylogenetic placement:**

```bash
# ROTL Python package
pip install rotl

python3 - <<'EOF'
import rotl
taxa = rotl.tnrs_match_names(["Lynx canadensis", "Lynx rufus", "Felis catus"])
for t in taxa:
    print(t.unique_name, t.ott_id, t.approximate_match)
EOF
```

### Handling Subspecies in Tree Labels

If tree tips are subspecies (`Mus musculus domesticus`) but assembly metadata
uses species-level (`Mus musculus`):
- Check NCBI Taxonomy for the subspecies taxid
- Use `ncbi-datasets` to list assemblies for the full lineage
- For VGP: check the ToLID metadata which records the exact individual collected

### Bulk Name Mapping Script

```bash
# Create mapping from tree tip labels to NCBI taxids
# Requires: requests, pandas

python3 - <<'EOF'
import requests, csv, time

ESEARCH = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi"
EFETCH  = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi"

def get_taxid(species):
    r = requests.get(ESEARCH, params={
        "db": "taxonomy", "term": f'"{species}"[All Names]',
        "retmode": "json", "retmax": 1
    })
    ids = r.json()["esearchresult"]["idlist"]
    return ids[0] if ids else None

with open("species_list.txt") as f, open("taxid_map.tsv", "w") as out:
    writer = csv.writer(out, delimiter="\t")
    writer.writerow(["species_name", "taxid"])
    for line in f:
        name = line.strip()
        taxid = get_taxid(name)
        writer.writerow([name, taxid or "NOT_FOUND"])
        time.sleep(0.34)  # Stay below 3 req/s unauthenticated rate limit
        print(f"{name} -> {taxid}")
EOF
```

---

## BED and Telomere Analysis

### Vertebrate Telomere Repeat

The canonical vertebrate telomere repeat is **TTAGGG** (5'→3' on the leading
strand). The complementary strand reads **CCCTAA**.

- Length: Typically 2–50 kb per chromosome end in mammals
- T2T (Telomere-to-Telomere): assembly resolves all the way to telomeric
  repeats at both ends of every chromosome

### Detecting Telomeric Windows with `tidk`

```bash
# Install tidk (Rust-based, fast)
cargo install tidk
# or conda install -c bioconda tidk

# Find the telomere repeat sequence (auto-detect or specify)
tidk explore --minimum 5 assembly.fa

# Generate telomere repeat coverage in BED windows
tidk search --string TTAGGG --window 10000 assembly.fa > telomere_windows.bed
# Output columns: chrom, start, end, forward_count, reverse_count

# Plot (requires matplotlib)
tidk plot --csv tidk_search.csv --output telomere_plot.pdf
```

### Sliding-Window k-mer Count (without tidk)

```bash
# Count TTAGGG in 10kb windows using bedtools and a custom script
bedtools makewindows -g assembly.fa.fai -w 10000 > windows.bed

bedtools getfasta -fi assembly.fa -bed windows.bed -fo windows.fa

# Count telomere repeat occurrences per window
python3 - <<'EOF'
import re
from Bio import SeqIO

for record in SeqIO.parse("windows.fa", "fasta"):
    seq = str(record.seq).upper()
    fwd = len(re.findall("TTAGGG", seq))
    rev = len(re.findall("CCCTAA", seq))
    # Parse "chr1:0-10000" header
    chrom, coords = record.id.rsplit(":", 1)
    start, end = coords.split("-")
    print(chrom, start, end, fwd, rev, sep="\t")
EOF
```

### Interpreting Telomere BED Coverage

**Chromosome-level assessment:**
- A chromosome with telomeric signal at **both ends** = likely T2T for that
  chromosome
- Signal only at one end = incomplete assembly at the other end
- No signal at either end = either: highly fragmented terminal regions, or
  genuine absence (centromeric chromosomes in some species)

**Expected density (vertebrates):**
- ~50–500 TTAGGG hexamers per 10 kb window in telomeric regions
- Background: < 5 per 10 kb in non-telomeric regions

```bash
# Identify chromosomes with signal at both ends
# Assumes first and last windows per chromosome are the ends
awk '{chrom=$1; start=$2; end=$3; fwd=$4; rev=$5}
     fwd+rev > 50 {print chrom, start}' telomere_windows.bed \
  | awk '{counts[$1]++} END{for(c in counts) if(counts[c]>=2) print c " has telomere signal"}'
```

### Intersecting Telomeres with AGP Boundaries

To check whether telomeric regions coincide with the ends of assembled
components (i.e. the assembly ends at a telomere, not at a gap):

```bash
# Convert AGP to BED for assembly component starts/ends
awk '$5 ~ /^[NU]$/ {print $1 "\t" $2-1 "\t" $3 "\tgap"}
     $5 !~ /^[NU]$/ {print $1 "\t" $2-1 "\t" $3 "\t" $6}' assembly.agp > components.bed

# Find the outermost components (first and last per chromosome)
sort -k1,1 -k2,2n components.bed | awk '
  !first[$1] {first[$1]=$0}
  {last[$1]=$0}
  END {for(c in first) {print first[c]; print last[c]}}
' > chromosome_ends.bed

# Intersect telomere windows with chromosome ends
bedtools intersect -a telomere_windows.bed -b chromosome_ends.bed -wa -wb \
  | awk '$4+$5 > 50' > telomere_at_ends.bed
```

---

## NCBI Data Integration

### NCBI Entrez API Basics

Rate limits:
- **Without API key:** max 3 requests/second
- **With API key:** max 10 requests/second
- **API key:** free, get at https://www.ncbi.nlm.nih.gov/account/

```python
from Bio import Entrez
Entrez.email = "your@email.com"
Entrez.api_key = "YOUR_API_KEY"  # Optional but recommended

# Always respect rate limits
import time
time.sleep(0.11)  # ~9 req/s with API key; 0.34s without
```

### Fetching Assembly Metadata

```python
from Bio import Entrez
import json

def get_assembly_metadata(assembly_accession):
    """Fetch assembly metadata from NCBI Assembly database."""
    handle = Entrez.esearch(db="assembly", term=assembly_accession)
    record = Entrez.read(handle)
    if not record["IdList"]:
        return None
    uid = record["IdList"][0]
    handle2 = Entrez.esummary(db="assembly", id=uid)
    summary = Entrez.read(handle2, validate=False)
    doc = summary["DocumentSummarySet"]["DocumentSummary"][0]
    return {
        "accession": doc.get("AssemblyAccession"),
        "name": doc.get("AssemblyName"),
        "organism": doc.get("Organism"),
        "taxid": doc.get("Taxid"),
        "contig_N50": doc.get("ContigN50"),
        "scaffold_N50": doc.get("ScaffoldN50"),
        "total_length": doc.get("TotalLength"),
        "chromosome_count": doc.get("Chrcount"),
        "level": doc.get("AssemblyStatus"),
        "submission_date": doc.get("SubmissionDate"),
    }
```

### Fetching Taxonomy Records

```python
def get_taxonomy(taxid):
    """Fetch full taxonomy record for a taxid."""
    handle = Entrez.efetch(db="taxonomy", id=str(taxid), retmode="xml")
    records = Entrez.read(handle)
    if not records:
        return None
    r = records[0]
    lineage = {rank["Rank"]: rank["ScientificName"]
               for rank in r.get("LineageEx", [])
               if rank["Rank"] != "no rank"}
    return {
        "taxid": r["TaxId"],
        "scientific_name": r["ScientificName"],
        "common_name": r.get("OtherNames", {}).get("CommonName", [""])[0],
        "rank": r["Rank"],
        "lineage": lineage,
    }
```

### NCBI Datasets CLI Tool

`datasets` is NCBI's command-line tool for bulk data downloads. Preferred over
the Entrez API for assembly downloads.

```bash
# Install
# Linux/macOS: download from https://ftp.ncbi.nlm.nih.gov/pub/datasets/command-line/
# or: conda install -c conda-forge ncbi-datasets-cli

# Download assembly FASTA by accession
datasets download genome accession GCF_018350175.1 \
  --include genome --filename mLynCan4.zip
unzip mLynCan4.zip

# Download all assemblies for a taxid (all birds)
datasets download genome taxon 8782 \
  --assembly-level chromosome \
  --assembly-source refseq \
  --include genome,gbff

# Summary of available assemblies for a species
datasets summary genome taxon "Lynx canadensis" \
  --assembly-level chromosome | python3 -m json.tool

# Get gene coordinates for a specific gene
datasets download gene symbol BRCA1 --taxon 9606 --include gene
```

### GenBank vs RefSeq Accession Conventions

| Prefix | Database | Meaning |
|--------|----------|---------|
| GCF_   | RefSeq   | NCBI-curated/annotated reference sequence |
| GCA_   | GenBank  | Submitter-provided assembly (may not be annotated) |
| NC_    | RefSeq   | Chromosome-level reference sequence |
| NW_    | RefSeq   | Unplaced/unlocalized scaffold |
| CM_    | GenBank  | Chromosome-level scaffold |

For VGP assemblies: both GCA_ (submitter original) and GCF_ (after NCBI
annotation) accessions exist for most species. Use GCF_ for analyses requiring
RefSeq annotation.

### Coordinate Mapping Between UCSC and Ensembl/NCBI

UCSC uses `chr1`-style names; Ensembl/NCBI use `1`-style names for human/mouse.
For other species, names may differ entirely.

```bash
# Download UCSC chromosome alias file for a genome
wget https://hgdownload.soe.ucsc.edu/goldenPath/hg38/database/chromAlias.txt.gz

# Or use the UCSC chrom alias table from NCBI
# This maps chr1 ↔ NC_000001.11 ↔ 1

# Convert a BED from UCSC names to Ensembl names
awk 'NR==FNR{map[$1]=$2; next} $1 in map{$1=map[$1]; print}' \
  ucsc_to_ensembl.tsv input.bed > output_ensembl.bed
```
