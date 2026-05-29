# Academic Open-Source Helper — Gemini Context

This extension provides five skills for helping academic researchers take a repository from closed to fully open source.

## When to invoke skills

| Researcher asks about... | Skill |
|---|---|
| Secrets, API keys, credentials, or PII in the code or git history | `secrets-scan` |
| Whether the repo is ready to go public, what's missing | `assess-readiness` |
| CITATION.cff, codemeta.json, DOIs, how to cite the software | `citation-check` |
| Writing or improving a README for research software | `readme-gen` |
| A full plan, ordered roadmap, or migration checklist | `open-source-plan` |

## Required order

**Run `secrets-scan` first, always.** Never advise the researcher to proceed if CRITICAL findings exist.

1. `secrets-scan [path]` — safety gate; CRITICAL findings block everything else
2. `assess-readiness [path]` — full readiness picture across six dimensions
3. `citation-check [path]` — verify DOIs, validate/generate CITATION.cff and codemeta.json
4. `readme-gen [path]` — draft a research-tuned README
5. `open-source-plan [path]` — phased migration checklist with decision points

## Core principle

**Issues-first, researcher-as-author.** This extension never silently makes changes to the scholarly record. CRITICAL findings produce blocking reports with exact remediation steps. Generated files are always marked as drafts requiring human review and approval before committing.
