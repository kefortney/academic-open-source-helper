---
name: academic-open-source-helper
description: Skills for taking an academic research project from closed to fully open source. Covers readiness assessment, secrets/PII scanning, citation integrity (DOI verification, CITATION.cff, codemeta), README generation, and migration planning. Use any skill in this plugin when helping a researcher publish, open-source, or improve the scholarly reproducibility of their code repository.
license: MIT
metadata:
  author: VERSO-UVM
  version: "1.0"
  homepage: https://github.com/VERSO-UVM/academic-open-source-helper
  standard: https://agentskills.io
---

This plugin provides five skills for academic open-source migration. Each can be invoked independently or as part of a full migration workflow.

## Skills

- `/assess-readiness` — scorecard of what's present, missing, or broken across all dimensions
- `/secrets-scan` — safety gate: secrets and PII in working tree and git history
- `/citation-check` — DOI verification, CITATION.cff, codemeta.json
- `/readme-gen` — draft a research-tuned README
- `/open-source-plan` — synthesize all findings into an ordered migration checklist

## Recommended order

1. `/secrets-scan` first — gate everything on this; never proceed if CRITICAL findings exist
2. `/assess-readiness` — full picture before deciding what to fix
3. `/citation-check` — the academic differentiator; fix before publishing
4. `/readme-gen` and `/open-source-plan` — documentation and roadmap last

## Philosophy

**Issues-first.** This plugin never silently authors changes to the scholarly record. CRITICAL findings produce a blocking issue description. ADDITIVE outputs (generated files) are clearly marked as drafts for human review. The researcher is always the author of every change.
