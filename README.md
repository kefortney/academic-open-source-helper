# academic-open-source-helper

Skills for helping academic researchers take a research repository from closed to fully open source. Compatible with Claude Code, Gemini CLI, OpenAI Codex, GitHub Copilot, and any agent supporting the [Agent Skills](https://agentskills.io) open standard.

## Skills

| Skill | Description |
| ----- | ----------- |
| `secrets-scan` | Safety gate — scans working tree and git history for secrets and PII |
| `assess-readiness` | Scorecard across safety, legal, citation, reproducibility, and documentation |
| `citation-check` | DOI verification, CITATION.cff validation/generation, codemeta.json |
| `readme-gen` | Drafts a research-tuned README with academic sections |
| `open-source-plan` | Synthesizes findings into a phased, decision-surfacing migration checklist |

## Recommended workflow

1. **`secrets-scan [path]`** — always first; gate everything on this result
2. **`assess-readiness [path]`** — full picture of what needs work
3. **`citation-check [path]`** — verify and generate citation metadata
4. **`readme-gen [path]`** — draft the README
5. **`open-source-plan [path]`** — generate the full migration checklist

## Installation

### Claude Code — as a plugin (namespaced)

```bash
git clone https://github.com/VERSO-UVM/academic-open-source-helper ~/.claude/plugins/academic-open-source-helper
```

Register in `~/.claude/settings.json`:

```json
{
  "plugins": ["~/.claude/plugins/academic-open-source-helper"]
}
```

Skills are available as `/academic-open-source-helper:secrets-scan`, etc.

### Claude Code — as personal skills (unnamespaced)

```bash
# Copy individual skills
cp -r skills/secrets-scan ~/.claude/skills/
cp -r skills/assess-readiness ~/.claude/skills/
# etc.

# Or symlink the whole directory
ln -s $(pwd)/skills/* ~/.claude/skills/
```

Skills are available as `/secrets-scan`, etc.

### Gemini CLI

```bash
git clone https://github.com/VERSO-UVM/academic-open-source-helper ~/.gemini/extensions/academic-open-source-helper
```

The `gemini-extension.json` manifest and `GEMINI.md` context file are included.

### OpenAI Codex

```bash
git clone https://github.com/VERSO-UVM/academic-open-source-helper
```

Point Codex to the `skills/` directory, or copy `AGENTS.md` to your project root for context. Individual skill directories follow the [Agent Skills](https://agentskills.io) standard and are read directly.

### Other agents (Goose, OpenHands, Roo Code, Junie, Amp, etc.)

Any agent supporting the [Agent Skills](https://agentskills.io) standard can load skills from the `skills/` directory directly. Clone the repo and configure your agent to read from `skills/`.

### GitHub Copilot

Copilot reads `.github/copilot-instructions.md` for project context. Copy that file to your research repository to give Copilot the full skill descriptions and recommended workflow. For full skills support, use a Copilot-compatible agent.

## Philosophy

**Issues-first, researcher-as-author.** This plugin never silently makes changes to the scholarly record. CRITICAL findings (secrets, PII, dead DOIs) are reported with exact remediation steps — the researcher acts, not the tool. Generated files (CITATION.cff, README drafts) are clearly marked as drafts for human review.

## Project context

Built by [VERSO](https://verso.uvm.edu) / ORCA at the University of Vermont. Part of a broader effort to make research software open-sourcing repeatable and teachable. See [`project_spec.md`](project_spec.md) for the full architecture including a planned MCP server layer.

## Standards

Skills follow established research-software standards: [CITATION.cff](https://citation-file-format.github.io/), [codemeta](https://codemeta.github.io/), [FAIR principles](https://www.go-fair.org/fair-principles/), [OSI licenses](https://opensource.org/licenses), and [Zenodo](https://zenodo.org/) for DOI minting. The plugin format follows the [Agent Skills](https://agentskills.io) open standard.
