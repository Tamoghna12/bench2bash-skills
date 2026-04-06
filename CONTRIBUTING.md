# Contributing to bench2bash-skills

Thank you for wanting to improve these skills. Contributions that make the content
more accurate, more complete, or more useful for learners are very welcome.

---

## Types of contributions

| Type | Examples | Welcome? |
|------|----------|----------|
| Content corrections | Wrong command, outdated tool version, incorrect benchmark | Yes |
| Coverage gaps | Missing domain, shallow section, missing tool | Yes |
| New skills | A new SKILL.md for a domain not yet covered | Yes (open an issue first) |
| Pedagogical improvements | Clearer explanation, better analogy, example interaction | Yes |
| Typos and formatting | Spelling, broken markdown tables | Yes |
| Unrelated to bioinformatics | General coding skills, unrelated domains | No |

---

## Before you start

For anything beyond a simple correction, **open an issue first** so the change
can be discussed before you invest time writing it. Use the appropriate issue
template:

- **Skill gap** — something is missing or too shallow
- **Bug report** — something is factually wrong
- **Feature request** — a new skill or major new section

---

## Making a change

1. **Fork** the repository and create a branch from `main`:
   ```bash
   git checkout -b fix/rnaseq-normalisation-explanation
   ```

2. **Edit the relevant file.** See [Repo structure](#repo-structure) below for
   where things live.

3. **Follow the style guide** (see below).

4. **Open a pull request** against `main`. Fill in the PR template.

---

## Style guide

### Skill files (`SKILL.md`)

- Written in second person ("when someone asks…", "help the learner…")
- No step-by-step commands in SKILL.md — those belong in `references/`
- Keep the description frontmatter under ~300 words
- Trigger phrases should be natural language users would actually say

### Reference files (`references/*.md`)

- Commands must be copy-paste runnable (no pseudocode placeholders)
- All tool flags should have brief inline explanations
- Tables are preferred over prose for: flag lists, comparison tables, benchmarks
- Code blocks must specify the language (` ```bash `, ` ```python `, ` ```r `)
- When a version matters (tool behaviour changed), note the version:
  `samtools ≥ 1.14`

### Content accuracy

- If you add a benchmark ("mapping rate should be > 95%"), cite the basis
  (tool documentation, published paper, or widely-accepted community standard)
- If you add a command, test it or note it is untested
- Use established tool names exactly as the tool authors spell them
  (e.g. `minimap2`, not `MiniMap2` or `Minimap2`)

---

## Repo structure

```
bench2bash-skills/
├── bioinformatics-tutor/
│   ├── SKILL.md                   # Pedagogical instructions for Claude
│   └── references/
│       ├── concepts.md            # Biological frameworks per domain
│       ├── learning-paths.md      # Learner progression by background
│       └── single-cell.md        # scRNA-seq / scATAC-seq teaching reference
│
├── bioinformatics-fundamentals/
│   ├── SKILL.md                   # Technical reference instructions
│   └── references/
│       ├── reference.md           # Format specs, tools, tech specs
│       ├── common-issues.md       # Troubleshooting guide
│       ├── genomeark-data-access.md
│       ├── genomic-analysis-patterns.md
│       └── workflow-reproducibility.md
│
└── releases/
    ├── bioinformatics-tutor.skill
    └── bioinformatics-fundamentals.skill
```

The `.skill` files in `releases/` are ZIP archives of the skill directory.
They are regenerated when content changes — do not edit them directly.

---

## Regenerating `.skill` files

After changing content, regenerate the packaged files:

```bash
# macOS / Linux
cd bench2bash-skills
rm -f releases/*.skill
zip -r releases/bioinformatics-tutor.skill bioinformatics-tutor/
zip -r releases/bioinformatics-fundamentals.skill bioinformatics-fundamentals/
```

Include the regenerated `.skill` files in your PR.

---

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).
By participating you agree to uphold it.

---

## Questions?

Open an issue or reach out via the [bench2bash newsletter](https://tamoghna12.github.io/bench2bash.github.io/).
