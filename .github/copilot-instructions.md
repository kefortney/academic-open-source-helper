# Academic Open-Source Helper

Skills for helping academic researchers take a repository from closed to fully open source. Follows the [Agent Skills](https://agentskills.io) open standard.

## Skills and when to use them

**`secrets-scan`** — Scan for hardcoded secrets, API keys, credentials, and PII in both the working tree and git history. Always run this first. Emit CRITICAL findings for anything that would be exposed on a public repo.

**`assess-readiness`** — Produce a scored checklist across six dimensions: safety, legal (LICENSE, dependency compatibility), academic metadata (CITATION.cff, codemeta, DOIs), reproducibility (pinned deps, environment spec, runnable example), documentation (README), and repository hygiene.

**`citation-check`** — Locate all DOIs in the repo and verify each one resolves via the Crossref REST API. Validate or generate CITATION.cff (v1.2.0) and codemeta.json. Check Zenodo archival readiness.

**`readme-gen`** — Draft a research-tuned README with academic sections: purpose, installation, usage, data provenance, data availability statement, methods/reproducibility, related publications (with DOI links), funding (with grant numbers), license, contributing, and contact. Never overwrite an existing README; always write to README_DRAFT.md and mark unfilled sections with `<!-- TODO -->`.

**`open-source-plan`** — Synthesize findings from the other skills into a six-phase migration checklist. Mark each item as `[ACTION]` (mechanical step) or `[YOUR DECISION]` (requires researcher judgment). Phase 1 (safety) blocks all later phases.

## Required order

1. `secrets-scan [path]` — **always first**; CRITICAL findings block everything else
2. `assess-readiness [path]`
3. `citation-check [path]`
4. `readme-gen [path]`
5. `open-source-plan [path]`

## Core principles

- **Issues-first**: Never silently make changes to the scholarly record. CRITICAL findings produce blocking reports.
- **Researcher-as-author**: Generated files (CITATION.cff, README drafts) are always marked as drafts for human review. The researcher makes every consequential decision.
- **Deterministic vs. prompt-backed**: Secrets scanning, DOI resolution, and schema validation are deterministic (same input → same output). README generation and license rationale are advisory drafts, never authoritative.

## Installation for use in your research repo

See [README.md](../README.md) for installation instructions for Claude Code, Gemini CLI, and other agents that support the Agent Skills standard.
