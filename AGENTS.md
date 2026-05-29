# Academic Open-Source Helper — Agent Context

This skill set helps academic researchers take a repository from closed to fully open source. It covers safety scanning, readiness assessment, citation integrity, README generation, and migration planning.

## Skills available

| Skill | Purpose |
|---|---|
| `secrets-scan` | Scan for hardcoded secrets, API keys, credentials, and PII in the working tree and git history |
| `assess-readiness` | Scored checklist across safety, legal, citation, reproducibility, and documentation |
| `citation-check` | Verify DOIs, validate or generate CITATION.cff and codemeta.json |
| `readme-gen` | Draft a research-tuned README with academic sections |
| `open-source-plan` | Phased migration checklist that surfaces decision points |

## Required order

**Run `secrets-scan` first.** Do not advise the researcher to proceed past a CRITICAL finding.

1. `secrets-scan [path]` — safety gate; CRITICAL findings block everything
2. `assess-readiness [path]` — full readiness picture
3. `citation-check [path]` — academic integrity check
4. `readme-gen [path]` — draft documentation
5. `open-source-plan [path]` — migration roadmap

Skills accept a path argument (e.g., `secrets-scan /path/to/repo`). The default is the current directory.

## Core principle

**Issues-first, researcher-as-author.** Never silently make changes to the scholarly record. Generated files (CITATION.cff drafts, README drafts) must be clearly marked for human review. The researcher makes every consequential decision; the skill surfaces options and findings.
