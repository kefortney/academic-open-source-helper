# academic-open-source-helper

A Claude Code skills plugin for helping academic researchers take a research repository from closed to fully open source.

## Skills

| Skill | Description |
| ----- | ----------- |
| `/assess-readiness` | Scorecard across safety, legal, citation, reproducibility, and documentation |
| `/secrets-scan` | Safety gate — scans working tree and git history for secrets and PII |
| `/citation-check` | DOI verification, CITATION.cff validation/generation, codemeta.json |
| `/readme-gen` | Drafts a research-tuned README with academic sections |
| `/open-source-plan` | Synthesizes findings into a phased, decision-surfacing migration checklist |

## Installation

### As a plugin (namespaced, no conflicts)

```bash
git clone https://github.com/VERSO-UVM/academic-open-source-helper ~/.claude/plugins/academic-open-source-helper
```

Then register it in your Claude Code settings (`~/.claude/settings.json`):

```json
{
  "plugins": ["~/.claude/plugins/academic-open-source-helper"]
}
```

Skills will be available as `/academic-open-source-helper:assess-readiness`, etc.

### As personal skills (unnamespaced)

Copy individual skills to your personal skills directory:

```bash
cp -r skills/assess-readiness ~/.claude/skills/
cp -r skills/secrets-scan ~/.claude/skills/
# etc.
```

Or symlink the whole directory:

```bash
ln -s $(pwd)/skills/* ~/.claude/skills/
```

## Recommended workflow

1. **`/secrets-scan [path]`** — always first; gate everything on this result
2. **`/assess-readiness [path]`** — full picture of what needs work
3. **`/citation-check [path]`** — verify and generate citation metadata
4. **`/readme-gen [path]`** — draft the README
5. **`/open-source-plan [path]`** — generate the full migration checklist

## Philosophy

**Issues-first, researcher-as-author.** This plugin never silently makes changes to the scholarly record. CRITICAL findings (secrets, PII, dead DOIs) are reported with exact remediation steps — the researcher acts, not the tool. Generated files (CITATION.cff, README drafts) are clearly marked as drafts for human review.

## Project context

Built by [VERSO](https://verso.uvm.edu) / ORCA at the University of Vermont. Part of a broader effort to make research software open-sourcing repeatable and teachable. See [`project_spec.md`](project_spec.md) for the full architecture including a planned MCP server layer.

## Standards

Skills follow established research-software standards: [CITATION.cff](https://citation-file-format.github.io/), [codemeta](https://codemeta.github.io/), [FAIR principles](https://www.go-fair.org/fair-principles/), [OSI licenses](https://opensource.org/licenses), and [Zenodo](https://zenodo.org/) for DOI minting.
