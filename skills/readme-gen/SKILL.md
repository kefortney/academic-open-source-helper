---
name: readme-gen
description: Draft a research-tuned README for an academic repository. Includes sections for purpose, installation, usage, data provenance, methods and reproducibility, how to cite, funding, related publications, and data availability. Use when a researcher needs a README that meets academic open-source best practices, not just a generic software README.
argument-hint: "[path-to-repo]"
context: fork
agent: claude
allowed-tools: Read Bash(find *) Bash(ls *) Bash(git log *) Bash(grep *) Write
---

Draft a research-tuned README for the repository at `$ARGUMENTS` (default: current working directory).

## Step 1 — Understand the project

Before writing, read the repository to understand what it actually does:

```bash
# What's there?
ls -la $REPO_PATH
find $REPO_PATH -maxdepth 2 -name "*.py" -o -name "*.R" -o -name "*.js" -o -name "*.ts" \
  -o -name "*.ipynb" | head -20

# Existing documentation
ls $REPO_PATH/{README*,CITATION*,codemeta.json,.zenodo.json} 2>/dev/null

# Detect language(s)
find $REPO_PATH -name "*.py" | head -1 && echo "Python"
find $REPO_PATH -name "*.R" | head -1 && echo "R"
find $REPO_PATH -name "*.jl" | head -1 && echo "Julia"
find $REPO_PATH -name "*.ipynb" | head -1 && echo "Jupyter"

# Dependencies
ls $REPO_PATH/{requirements*.txt,setup.py,pyproject.toml,DESCRIPTION,package.json,Gemfile} 2>/dev/null

# Funding/grant info in existing files
grep -rni "grant\|funding\|nsf\|nih\|doe\|nasa\|award" $REPO_PATH --include="*.md" --include="*.cff" --include="*.py" | head -10

# Authors from git
git -C $REPO_PATH log --format='%aN' | sort | uniq -c | sort -rn | head -10
```

Read the main entry point or primary script to understand what the code does.

## Step 2 — Draft the README

Write the README to `$REPO_PATH/README.md` (or `README_DRAFT.md` if a README already exists — never overwrite silently).

Use this section structure, adapting to what you actually found. Mark any section where you don't have enough information with `<!-- TODO: fill in -->`.

---

```markdown
# [Project Name]

[One-sentence description of what this software does and what research problem it addresses.]

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
<!-- Add DOI badge here once Zenodo-archived: [![DOI](https://zenodo.org/badge/...)] -->

## Overview

[2–3 sentences: what the software does, what data or inputs it works with, and what outputs or findings it produces. This is for someone who landed here from a citation.]

## Citation

If you use this software, please cite:

```bibtex
<!-- TODO: Add BibTeX entry, or reference CITATION.cff -->
```

See [`CITATION.cff`](CITATION.cff) for full citation metadata.

## Installation

```bash
# [language-appropriate install instructions]
```

**Requirements:** [Python 3.x / R x.x / etc.], [key dependencies]

## Usage

```bash
# Minimal working example
```

[Brief explanation of key parameters or configuration.]

## Data

### Input data
[Where input data comes from — download URL, DOI, or "included in repo". If data is not included, explain why and how to obtain it.]

### Data availability statement
[Standard academic statement: e.g., "Data used in this study is available at [DOI/URL] under [license]." or "Data cannot be shared publicly due to [IRB/privacy/agreement]; synthetic examples are included in `data/example/`."]

## Methods and Reproducibility

[Brief description of the core algorithm or analytical approach, with reference to the associated paper if applicable.]

To reproduce the main results from [paper/analysis]:

```bash
# Step-by-step reproduction commands
```

See [`REPRODUCE.md`](REPRODUCE.md) or the [methods notebook](notebooks/) for full details.
<!-- TODO: confirm reproduction path exists -->

## Repository Structure

```
[project-name]/
├── [main module or script]
├── data/           # [description]
├── notebooks/      # [description, if applicable]
├── tests/          # [description]
└── docs/           # [description, if applicable]
```

## Related Publications

<!-- TODO: Add DOI links for associated papers -->
- [Author et al., Year. "Title." *Journal*. https://doi.org/...]

## Funding

<!-- TODO: fill in grant numbers and agencies -->
This work was supported by [Agency] grant [number] to [PI name].

## License

[License name] — see [`LICENSE`](LICENSE) for details.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for guidelines.
<!-- TODO: create CONTRIBUTING.md if it doesn't exist -->

## Contact

[Name], [Institution] — [email or GitHub handle]
```

---

## Step 3 — Report what you filled in vs. what needs human review

After writing the draft, produce a summary:

```
## README Draft: [repo name]

**Output:** README_DRAFT.md (review before renaming to README.md)

### Filled in from repo content
- [list sections you populated with real data]

### Needs human input (marked <!-- TODO --> in draft)
- [list sections that need the researcher's knowledge]
  - e.g., "Data availability statement — needs IRB/data-sharing decision"
  - e.g., "Funding — needs grant numbers"
  - e.g., "Related publications — needs DOIs for associated papers"

### Sections omitted (not applicable)
- [e.g., "Reproduction section — no scripts found; add when code is complete"]
```

Keep the draft honest: it is better to have explicit `<!-- TODO -->` markers than confident-sounding placeholder text that a researcher might not notice and accidentally publish.
