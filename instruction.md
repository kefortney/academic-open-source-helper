# Claude Code Skills Library — Research Notes

> Context: Exploring how to build a public GitHub repo of Claude Code skills for VERSO/ORCA use, shareable with Vermont agency partners.

## Key Concepts

- **Skills** are the current format (supersedes `.claude/commands/`)
- Each skill is a directory with a `SKILL.md` entrypoint
- Claude can invoke skills automatically based on `description`, or you invoke manually with `/skill-name`

---

## Directory Structure

### Single Skill
```
my-skill/
├── SKILL.md           # Required — main instructions + frontmatter
├── reference.md       # Optional — detailed docs, loaded on demand
├── examples/
│   └── sample.md      # Optional — example output
└── scripts/
    └── helper.py      # Optional — scripts Claude can run
```

### Scope / Where Skills Live
| Scope | Path | Applies to |
|---|---|---|
| Personal | `~/.claude/skills/<skill-name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<skill-name>/SKILL.md` | This project only |
| Plugin | `<plugin>/skills/<skill-name>/SKILL.md` | Where plugin is enabled |

---

## SKILL.md Format

```yaml
---
name: my-skill                        # Display name (optional, defaults to dir name)
description: What it does and when.   # STRONGLY recommended — drives auto-invocation
disable-model-invocation: true        # true = user-only, Claude won't auto-trigger
allowed-tools: Bash(git add *) Bash(git status *)
context: fork                         # Run in isolated subagent
agent: Explore                        # Which subagent type (with context: fork)
argument-hint: "[issue-number]"       # Shown in autocomplete
paths: "**/*.py"                      # Only activate for matching files
---

## Instructions

Write instructions here. Reference supporting files so Claude knows they exist:
- For API details, see [reference.md](reference.md)
```

### Key Frontmatter Fields
| Field | Notes |
|---|---|
| `description` | Most important — Claude uses this to decide when to load the skill |
| `disable-model-invocation` | `true` = manual `/skill-name` only, won't auto-trigger |
| `user-invocable` | `false` = Claude-only, hidden from `/` menu |
| `allowed-tools` | Pre-approve tools so Claude doesn't prompt per-use |
| `context: fork` | Runs skill in isolated subagent (clean context) |
| `paths` | Glob patterns — only activate for matching files |

### Dynamic Context Injection
Run shell commands before Claude sees the skill content:
```markdown
## Current diff
!`git diff HEAD`

## Environment
```!
node --version
git status --short
```
```

### Arguments
```yaml
---
name: fix-issue
argument-hint: "[issue-number]"
---
Fix GitHub issue $ARGUMENTS following our coding standards.
```
Invoked as `/fix-issue 123` → `$ARGUMENTS` = `123`

---

## Repo Structure Options for VERSO

### Option A: Plugin (recommended for distribution)
Namespaces all skills as `/verso:skill-name` — no conflicts with user's personal skills.
```
verso-claude-skills/          # GitHub repo
├── README.md
├── SKILL.md                  # Plugin root (optional)
└── skills/
    ├── gis-analysis/
    │   └── SKILL.md
    ├── open-source-pr-review/
    │   └── SKILL.md
    ├── orca-project-setup/
    │   └── SKILL.md
    └── vt-data-standards/
        └── SKILL.md
```
Install: clone repo, register as plugin in Claude Code settings.

### Option B: Flat skills repo (simpler, users copy what they want)
```
verso-claude-skills/          # GitHub repo
├── README.md
├── gis-analysis/
│   └── SKILL.md
├── open-source-pr-review/
│   └── SKILL.md
└── orca-project-setup/
    └── SKILL.md
```
Install: `cp -r gis-analysis ~/.claude/skills/` or symlink the whole dir.

---

## GitHub Copilot Compatibility

**Short answer: not natively compatible.** Different formats.

| Tool | Format |
|---|---|
| Claude Code | `.claude/skills/<name>/SKILL.md` (plugin or personal) |
| GitHub Copilot | `.github/copilot-instructions.md` (single flat file) |

Claude Code skills follow the [Agent Skills open standard](https://agentskills.io) — worth checking if Copilot has adopted it.

**Practical approach:** Write in Claude Skills format (more expressive), maintain a flattened `.github/copilot-instructions.md` as a secondary artifact.

---

## Useful Commands in Claude Code
| Command | Purpose |
|---|---|
| `/skills` | List available skills, toggle visibility |
| `/skill-name` | Invoke a skill directly |
| `/doctor` | Check if skill descriptions are being truncated due to budget |
| `/memory` | Edit CLAUDE.md files |

---

## Official Docs
- Skills: https://code.claude.com/docs/en/slash-commands
- Plugins: https://code.claude.com/docs/en/plugins
- Subagents: https://code.claude.com/docs/en/sub-agents
- Agent Skills standard: https://agentskills.io

