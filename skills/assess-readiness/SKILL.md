---
name: assess-readiness
description: Assess whether an academic research repository is ready to be open-sourced. Produces a scored checklist covering LICENSE, README, citation metadata, reproducibility, secrets, and PII. Use when a researcher wants to know what work is needed before publishing their code.
license: MIT
compatibility: Requires git. The path argument replaces $ARGUMENTS (default: current directory).
metadata:
  author: VERSO-UVM
  version: "1.0"
argument-hint: "[path-to-repo]"
context: fork
agent: claude
allowed-tools: Read Bash(find *) Bash(ls *) Bash(git log *) Bash(git diff *) Bash(grep *)
---

Assess the academic open-source readiness of the repository at `$ARGUMENTS` (default: current working directory).

## What to check

Work through each dimension in order. For each item, emit one of: ✅ present/passing, ⚠️ present but incomplete, ❌ missing/failing, 🔴 CRITICAL (blocks open-sourcing).

### 1. Safety (gate — check first)
- [ ] No secrets or API keys in working tree (grep for common patterns: `sk-`, `ghp_`, `AKIA`, `-----BEGIN`, `password =`, `token =`)
- [ ] No secrets in git history (check recent commits with `git log -p --all -- '*.env' '*.key' '*.pem'`)
- [ ] No PII in data files (SSN patterns `\d{3}-\d{2}-\d{4}`, bare email lists in CSV/TSV/JSON)

**If any Safety item is 🔴 CRITICAL, stop and report. Do not proceed to other dimensions until the researcher resolves it.**

### 2. Legal
- [ ] LICENSE file exists in repo root
- [ ] License is OSI-approved (MIT, Apache-2.0, GPL-2.0+, BSD-2-Clause, BSD-3-Clause are common)
- [ ] No dependency with a more restrictive license than the project license (GPL contamination)
- [ ] No copyright notices pointing to a third party without a clear license grant

### 3. Academic metadata
- [ ] `CITATION.cff` exists and is valid YAML
- [ ] `codemeta.json` exists
- [ ] All DOIs in README/CITATION.cff are present (flag any `https://doi.org/` links for the citation-check skill to verify)
- [ ] ORCID IDs for authors, if applicable

### 4. Reproducibility (FAIR)
- [ ] Dependencies are pinned or have a lockfile (`requirements.txt`, `poetry.lock`, `Pipfile.lock`, `environment.yml`, `renv.lock`, `package-lock.json`)
- [ ] An environment spec exists (`environment.yml`, `Dockerfile`, `devcontainer.json`)
- [ ] A runnable example exists (script, notebook, or documented quickstart)
- [ ] Data availability statement in README (where to get input data, or that data is included)

### 5. Documentation
- [ ] `README.md` exists
- [ ] README contains: purpose/description, installation, usage, how to cite, license section
- [ ] `CONTRIBUTING.md` exists (optional but good practice)

### 6. Repository hygiene
- [ ] `.gitignore` exists and covers language-appropriate patterns
- [ ] No large binary files committed (>1 MB non-data files)
- [ ] No untracked sensitive files (`.env`, `*.pem`, `secrets.yaml`)

## Output format

Produce a scorecard in this format:

```
## Open-Source Readiness: [repo name]

### Summary
- Total items: N
- ✅ Passing: N
- ⚠️ Needs work: N
- ❌ Missing: N
- 🔴 CRITICAL (blocks): N

**Readiness level:** [Not ready / Needs work / Nearly ready / Ready]

### Safety
[findings]

### Legal
[findings]

### Academic Metadata
[findings]

### Reproducibility
[findings]

### Documentation
[findings]

### Repository Hygiene
[findings]

### Recommended next steps (in order)
1. [CRITICAL items first, then ❌ missing, then ⚠️ improvements]
```

Be specific: include file paths, line numbers, and exact remediation steps where known. For CRITICAL findings, include the exact command or action needed (e.g., `git filter-repo --path secrets.env --invert-paths`).
