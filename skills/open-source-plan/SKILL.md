---
name: open-source-plan
description: Generate a comprehensive, ordered migration plan to open-source an academic research repository. Synthesizes readiness findings, safety concerns, licensing, citation integrity gaps, and documentation into a human-reviewable checklist that surfaces decision points rather than making them automatically. Use when a researcher wants a full roadmap before beginning the open-sourcing process.
argument-hint: "[path-to-repo]"
context: fork
agent: claude
allowed-tools: Read Bash(find *) Bash(ls *) Bash(git log *) Bash(grep *) Write
---

Generate a migration plan for open-sourcing the repository at `$ARGUMENTS` (default: current working directory).

**This skill synthesizes findings — it does not run the full scans itself.** If you have output from `/assess-readiness`, `/secrets-scan`, or `/citation-check` in the conversation, use that. Otherwise, do a quick survey of the repository to identify the most obvious issues before generating the plan.

## Step 1 — Quick survey (if no prior findings in context)

```bash
# Safety quick-check
grep -rn --include="*.py" --include="*.js" --include="*.env" --include="*.yml" \
  -E "(password|api_key|secret|token)\s*[=:]\s*['\"][^'\"]{8,}" $REPO_PATH 2>/dev/null | head -5

# License
ls $REPO_PATH/LICENSE* 2>/dev/null || echo "NO LICENSE"

# Citation metadata
ls $REPO_PATH/{CITATION.cff,codemeta.json} 2>/dev/null

# README
ls $REPO_PATH/README* 2>/dev/null || echo "NO README"

# Dependencies pinned?
ls $REPO_PATH/{requirements*.txt,poetry.lock,Pipfile.lock,environment.yml,renv.lock,package-lock.json} 2>/dev/null || echo "NO LOCKFILE"

# Recent commits (understand state of work)
git -C $REPO_PATH log --oneline -10 2>/dev/null
```

## Step 2 — Generate the migration plan

Write the plan to stdout (and optionally to `OPEN_SOURCE_PLAN.md` in the repo root if the user wants a file).

Structure the plan in strict priority order. Items block later items; do not reorder.

---

```markdown
# Open-Source Migration Plan: [repo name]

Generated: [date]
Repository: [path or URL]
Assessed by: Claude ([skill version context])

---

## ⚠️ How to use this plan

This plan surfaces what needs to happen and where decisions belong to you — it does not make decisions for you. Items marked **[YOUR DECISION]** require your judgment or information only you have. Items marked **[ACTION]** have a clear mechanical step.

Work through sections in order. Do not proceed past a 🔴 CRITICAL item.

---

## Phase 1 — Safety (MUST complete before anything else)

🔴 = blocks open-sourcing entirely; do not skip.

- [ ] 🔴 [ACTION] Run a full secrets scan: `/secrets-scan [path]`
  - If findings: rotate any exposed credentials immediately (treat as compromised)
  - Remove from history with `git filter-repo` (destructive; coordinate with collaborators)
  - Re-run scan to confirm clean
- [ ] 🔴 [YOUR DECISION] Data files: do any contain participant PII, unpublished data, or data under a data-use agreement that prohibits redistribution?
  - If yes: remove or replace with synthetic examples before publishing
- [ ] 🔴 [YOUR DECISION] Institutional review: does your IRB approval, data-sharing agreement, or funder terms permit open publication of this code and any bundled data?

**Do not proceed to Phase 2 until all Phase 1 items are resolved.**

---

## Phase 2 — Legal

- [ ] [ACTION/YOUR DECISION] Add a LICENSE file
  - If federally funded (NSF, NIH, DOE, NASA): consider whether funder open-access requirements apply
  - If no dependencies with restrictive licenses: MIT or Apache-2.0 are good defaults
  - If dependencies include GPL: your license must be GPL-compatible (GPL-3.0 or later)
  - **[YOUR DECISION]** Choose your license — this is a legal decision, not a technical one
  - [ACTION] Copy the license text from https://choosealicense.com/ into `LICENSE`
- [ ] [ACTION] Verify dependency licenses don't contaminate your chosen license
  - Python: `pip-licenses` or check each package on PyPI
  - R: check CRAN license fields
  - Node: `license-checker`

---

## Phase 3 — Academic metadata (the differentiator)

- [ ] [ACTION] Verify all DOIs in README/citations resolve: `/citation-check [path]`
  - Dead DOIs must be corrected before publishing — they are permanent in the scholarly record
  - Mismatched DOIs (pointing to the wrong paper) are a citation integrity issue
- [ ] [ACTION/YOUR DECISION] Add `CITATION.cff`
  - [ACTION] Draft with `/citation-check` or from template at https://citation-file-format.github.io/
  - **[YOUR DECISION]** Confirm author list, ORCID IDs, and grant numbers — only you know these
  - [ACTION] Validate: `cffconvert --validate -i CITATION.cff`
- [ ] [ACTION/YOUR DECISION] Add `codemeta.json` (research software metadata)
  - Converts automatically from CITATION.cff: `cffconvert --format codemeta -i CITATION.cff > codemeta.json`
- [ ] [YOUR DECISION] Zenodo archival: do you want a persistent DOI for the software itself?
  - If yes: connect repo to Zenodo (https://zenodo.org/account/settings/github/), create a release, Zenodo will mint a DOI
  - Add `DOI` to CITATION.cff after minting

---

## Phase 4 — Reproducibility

- [ ] [ACTION] Pin dependencies
  - Python: `pip freeze > requirements.txt` or use `poetry`/`pipenv` for a lockfile
  - R: `renv::snapshot()`
  - Node: commit `package-lock.json`
- [ ] [YOUR DECISION] Environment specification: is there a `Dockerfile`, `environment.yml`, or `devcontainer.json`?
  - Recommended for computational results that need to be reproducible beyond dependency pinning
- [ ] [YOUR DECISION] Data availability: what is the statement for your README?
  - Options: data included in repo / data available at [DOI] / data cannot be shared (state why) / synthetic examples included
- [ ] [ACTION] Runnable example: can someone clone this and run something in <5 minutes?
  - If not, add a minimal example script or notebook

---

## Phase 5 — Documentation

- [ ] [ACTION] Add or update README using `/readme-gen [path]`
  - Required sections: purpose, installation, usage, citation, license
  - Academic sections: data provenance, reproducibility, funding, related publications
- [ ] [ACTION] Add `.gitignore` if missing (https://gitignore.io for language-appropriate template)
- [ ] [OPTIONAL] Add `CONTRIBUTING.md` if you want external contributions

---

## Phase 6 — Final checks before publishing

- [ ] [ACTION] Re-run `/assess-readiness [path]` — should show all ✅ except optional items
- [ ] [ACTION] Make the repository public on GitHub (Settings → Change visibility)
- [ ] [ACTION] Create a GitHub release with a version tag (e.g., `v1.0.0`)
  - If connected to Zenodo, this triggers automatic DOI minting
- [ ] [ACTION] Update CITATION.cff with the Zenodo DOI once minted
- [ ] [YOUR DECISION] Announce: where does your community expect to hear about new software releases? (JOSS, rOpenSci, domain-specific lists, lab website)

---

## Decision log (fill in as you work)

| Decision | Your choice | Date |
|----------|-------------|------|
| License | | |
| Data availability statement | | |
| Zenodo archival | yes / no | |
| PII/IRB cleared | yes / deferred | |

---

## Items that need more information before this plan can be completed

<!-- List anything the quick survey couldn't determine -->
```

---

After generating the plan, note explicitly:

1. **What was inferred from the repo** vs. **what is a placeholder** (so the researcher knows what's a real finding vs. a template item).
2. **The single most important action right now** (usually Phase 1 safety, or adding a LICENSE if no safety issues found).
3. **Estimated effort** if you can gauge it: "This looks like 2–3 hours of work, mostly filling in metadata" vs. "The lack of a lockfile and missing data-availability information will require significant decisions."
