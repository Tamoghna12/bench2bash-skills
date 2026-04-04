# Learning Paths by Background

Structured progressions for three learner types. These are not rigid curricula —
they're maps of where to start and where the natural next questions lead. Adapt
to what the learner is actually working on.

---

## Path 1: Wet Lab Biologist → Bioinformatics

**Starting point:** Strong biology, zero or minimal coding experience.
**Goal:** Be able to run standard analyses on their own data and interpret the
biology with confidence.

### Stage 1: The Terminal Is Just a Lab Bench (Week 1–2)

The anxiety is the terminal. Defuse it first.

- Teach: what a file system is (analogy: a filing cabinet with folders)
- `ls`, `cd`, `mkdir`, `cp`, `mv`, `cat`, `less` — the "handling" commands
- Text files vs binary files — why you can read a FASTQ but not a BAM
- What a script is: a recipe you write once and run many times

**Biological bridge:** The terminal is how you instruct a very precise, very
fast lab assistant. Every command is like a protocol step. Unlike a pipette, it
will do exactly what you say, every time — which means your mistakes are also
precise.

### Stage 2: Your Data Has a Shape (Week 2–4)

- What is a FASTQ file? (Your sequencer's output: reads + quality scores)
- What is a FASTA file? (Sequences without quality scores — reference genomes,
  assembled contigs)
- What is a BAM file? (Aligned reads: where did each fragment come from?)
- What is a VCF? (Variants: where does my sample differ from the reference?)
- What is a BED file? (Coordinates: mark regions of the genome)

**Biological bridge:** Think of FASTQ as the raw readout from your sequencer —
like the raw signal from a flow cytometer before you apply gates. The reference
genome is the gating strategy: it tells you how to interpret the signal.

### Stage 3: The Standard Pipelines (Month 1–2)

Work through these in order because each builds on the last:

1. **Quality control:** FastQC → understand what "good" and "bad" reads look like
2. **Trimming:** Trim adapters and low-quality ends (Trimmomatic, fastp)
3. **Alignment:** Map reads to reference (BWA-MEM2 for DNA, STAR for RNA)
4. **Quantification / variant calling:** depends on experiment type
5. **Interpretation:** what does the biology say?

At each step: teach not just the command but *why* the step is necessary. What
goes wrong if you skip adapter trimming? (Adapters align to the genome in weird
places, creating false-positive variants and inflated counts near gene ends.)

### Stage 4: Biological Validation Reflexes (Ongoing)

At this point, the learner can run the pipeline. The next challenge is
*trusting* the output. Build these habits:

- Always check FastQC before anything else
- Always run `samtools flagstat` after alignment and eyeball the numbers
- Always ask "what would a good result look like here?" before looking at output
- Always have a positive control: a gene/variant you *know* should be there

---

## Path 2: CS / Data Scientist → Bioinformatics

**Starting point:** Strong coding, data fluency, zero biology.
**Goal:** Understand enough biology to ask good questions, design valid analyses,
and interpret results in biological terms.

### Stage 1: What Is a Genome, Really? (Week 1)

Don't start with tools. Start with reality.

- A genome is a physical molecule: DNA, double-stranded, made of four bases
- The sequence encodes information — but not as simply as a program
- Genes are ~1–2% of the human genome; the rest has regulatory, structural, and
  unknown functions
- Alternative splicing: one gene can produce many proteins
- Not all "DNA" in a cell is nuclear: mitochondria have their own genome

**Computational translation:** The genome is a string, but one with rich
annotation layered on top — gene models, regulatory regions, repeats. It's more
like a heavily annotated text than a clean data schema.

### Stage 2: Why Biology Is Messier Than Code (Week 2–3)

- Biological variation is continuous and multidimensional, not categorical
- The "reference genome" is a construct — no individual has exactly that sequence
- Gene expression is context-dependent: the same genome in different cell types
  produces radically different transcriptomes
- Measurement is inherently noisy: PCR amplification, library prep bias,
  sequencing error
- Correlation is not causation: differentially expressed genes are associated
  with a condition, not necessarily causal

**The key lesson:** You can write code that processes biological data correctly
and still produce biologically wrong conclusions if you don't understand what
the data represents.

### Stage 3: Data Fluency in a Biological Context

For someone already comfortable with data: map biological data formats to
familiar concepts.

| Bioinformatics concept | Data equivalent |
|---|---|
| FASTQ file | List of (string, quality_array) tuples |
| Genome (FASTA) | One very long string, or a dict of chromosome → string |
| BAM file | Sorted, indexed database of read alignments |
| VCF | DataFrame of variants: position, ref_allele, alt_allele, genotype, quality |
| Count matrix (RNA-seq) | Feature × sample matrix of integers |
| BED file | DataFrame of genomic intervals: chr, start, end |

The computational operations make sense from first principles — alignment is
approximate string matching; variant calling is Bayesian model comparison;
differential expression is a GLM on count data. What's new is *why* each
operation matters biologically.

### Stage 4: Biological Sanity Checks as Code Review

Teach the CS person to review biological outputs the way they'd review code:

- Every result should have a "does this make sense?" test
- Known positive controls: if this gene should be expressed, is it?
- Order-of-magnitude checks: does the scale of this result match biological
  reality?
- Reproducibility: would this result change if I reran with a different seed
  / parameter / batch?

---

## Path 3: Early PhD Student (Mixed Background)

**Starting point:** Has some biology AND some coding but lacks integration.
Can run tools but not sure when to trust them.

### The Core Gap

Early PhD students often know *how* to run analyses but not *what to look for*.
They get a result and don't know if it's good or garbage. The focus here is on
developing that reference frame.

### Stage 1: Know What "Good" Looks Like

For every analysis type you routinely do, build a mental reference:

- What N50 should this organism's assembly have? (For bacteria: > 100 kb for
  Illumina, > 1 Mb for HiFi. For mammals: > 10 Mb for chromosome-scale.)
- What mapping rate should I expect? (Varies by tech — know the norms for yours)
- What does a good PCA look like for my experiment?
- What does my data look like when the experiment worked vs failed?

This comes from reading, from talking to people, and from accumulating examples.
This skill can scaffold it by asking the learner to benchmark their expectations
before looking at results.

### Stage 2: Diagnose, Don't Just Run

When something looks wrong, don't just rerun the tool. Ask:

1. Is this a data quality problem? (Check FastQC, flagstat, coverage)
2. Is this a parameter choice problem? (Read the tool documentation — what does
   the default parameter assume?)
3. Is this an experimental design problem? (Is there a confound? A batch effect?)
4. Is this actually real biology that surprises me? (The most exciting option)

### Stage 3: Develop Domain-Specific Intuition

This is where the biological eye becomes field-specific. For a bacterial
pangenomics PhD student, the questions become:

- Is this accessory gene genuinely variable across strains, or is it an assembly
  artefact?
- Does this biosynthetic cluster occur in multiple lineages, or was it transferred?
- Does my metabolic model predict growth in conditions where the strain grows in
  the lab?
- Is this pangenome open or closed, and does that match what I know about the
  ecology of this organism?

The skill should push learners to formulate these questions themselves, then
help them find the tools and data to answer them.

---

## Key Resources Worth Knowing

**For wet lab biologists learning computation:**
- *Bioinformatics Data Skills* (Buffalo) — learn the Unix + Python foundation
- *Bioinformatics Algorithms* (Compeau & Pevzner) — deep conceptual grounding

**For data scientists learning biology:**
- *Molecular Biology of the Cell* — not optional; understand what you're modelling
- *An Introduction to Bioinformatics Algorithms* — bridges CS to biology

**For everyone:**
- EMBL-EBI training courses (free online) — practical, biology-first
- Bioinformatics Stack Exchange — "why is my result weird?" community
- The tool documentation — read the paper, not just the README
