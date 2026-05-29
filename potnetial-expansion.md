# Skill-Backed MCP: Open-Sourcing Academic Research

**A VERSO / ORCA project specification — v2**
**Author:** Kendall Fortney, Director, VERSO (UVM) with Claude Opus
**Status:** Draft for scoping

---

## 1. Concept

Anthropic's "skills" (`SKILL.md` files) are prompt documents: structured instructions that tell a model *how to perform a task*. They contain no executable code and no input/output schema. They shape behavior, not computation.

MCP (Model Context Protocol) is the opposite: a transport-and-discovery protocol that lets a model call external tools with defined schemas and get structured results back. MCP standardizes the *wire*; it doesn't care what the tool does.

The two never meet directly — you can't expose a markdown instruction file as an MCP tool, because there's nothing to call. **This project builds the missing layer between them**: a single MCP server exposing a curated set of capabilities as callable tools. The caller — a PI, an ORCA student, a partner — calls `assess_repo_readiness(path)` and gets a structured report, never touching the underlying prompt engineering or API mechanics.

The chosen domain is **taking an academic research project from closed to fully open source**, following established research-software best practices (FAIR, OSI licensing, reproducibility, CITATION.cff, codemeta, DOI verification). This is VERSO's existing domain, the workflow is well-defined enough to encode, and the academic layer is the differentiator: generic open-sourcing tools handle LICENSE and README; none handle citation integrity, research metadata standards, or reproducibility for the scholarly record.

### Design principle: modularity above all

The single most important architectural commitment in v2 is that **this is a plugin system, not a monolith.** The core mechanics — tool discovery, dispatch, the GitHub output layer, the findings model — must be completely decoupled from any individual capability. Adding "verify DOIs" or "lint R code" must never require touching the core. If adding a capability means editing the engine, the architecture has failed.

Everything below is built so that a capability is a self-contained, droppable unit and the core neither knows nor cares what capabilities exist until startup.

### Why build it
- **Reusability.** Open-sourcing research is a repeatable ~10-step procedure. Encode once, run for every partner.
- **Academic differentiation.** DOI verification, codemeta, citation integrity — verifiable, fact-based, and skipped by every generic tool.
- **Lowered barrier.** Researchers who'd never write a prompt call a tool that does the right thing by default.
- **Teaches the pattern.** ORCA students learn MCP, schema design, plugin architecture, and CI on a real deliverable.

### Where it goes wrong (read before committing)
- **Prompt-backed tools are non-deterministic.** Anything touching correctness (secrets, license-as-fact, DOI resolution, schema validity) must be deterministic code, never a model call.
- **Tool sprawl.** Modularity makes adding tools easy — which makes over-adding tools easy. A calling model with 30 tools makes bad choices. Curate the active set.
- **Maintenance.** Standards drift (licenses, metadata schemas). Versioned, owned skills, or it rots.
- **Scope.** v2 is already much bigger than v1. Sequence ruthlessly; ship the verifiable core before the judgment features.
- **Provenance.** This touches the scholarly record. The tool must never silently author changes — see the issues-first model in §4.

---

## 2. Architecture

### Core principle: thin engine, fat plugins

```
┌─────────────┐   JSON-RPC (stdio/SSE)   ┌────────────────────────────┐
│ MCP Client  │◄────────────────────────►│      Skill-MCP Engine      │
│ (Claude     │                          │  (the CORE — rarely changes)│
│  Desktop,   │                          │                            │
│  Inspector, │                          │  • plugin loader/registry  │
│  partner    │                          │  • dispatch + validation   │
│  app)       │                          │  • Findings model          │
└─────────────┘                          │  • GitHub output adapter   │
                                         │  • context provider         │
                                         └──────────────┬─────────────┘
                                                        │ discovers at startup
                                         ┌──────────────▼─────────────┐
                                         │      CAPABILITY PLUGINS     │
                                         │   (drop-in, self-contained) │
                                         │                            │
                                         │  readiness/   secrets/      │
                                         │  license/     citation_doi/ │
                                         │  readme/      codemeta/      │
                                         │  reproduce/   lint/          │
                                         │  docs/        github_issues/ │
                                         └──────────────┬─────────────┘
                                                        │ operate on
                                         ┌──────────────▼─────────────┐
                                         │   WORK CONTEXT (the input)  │
                                         │  • local path to cloned repo│
                                         │  • repo URL + auth (opt)    │
                                         │  • findings accumulator     │
                                         └────────────────────────────┘
```

### The plugin contract (the heart of modularity)

Every capability is a folder under `plugins/`. The engine discovers it at startup and needs nothing else. A plugin is:

```
plugins/
  citation_doi/
    manifest.json     # name, version, tool schema, dependencies, kind
    SKILL.md          # instructions (only if prompt-backed)
    handler.py        # implements a single contract function
    tests/            # the plugin owns its own tests + fixtures
```

The contract is one function with a fixed signature. Every plugin implements:

```python
def run(ctx: WorkContext, params: dict) -> ToolResult:
    # ctx exposes: ctx.repo_path, ctx.repo_url, ctx.add_finding(...),
    #              ctx.claude(prompt) for prompt-backed plugins,
    #              ctx.read_skill() to load this plugin's SKILL.md
    ...
    return ToolResult(data=..., findings=[...])
```

The engine guarantees the `WorkContext`. The plugin guarantees it touches nothing outside `ctx`. That boundary is what keeps capabilities from breaking the core or each other. A new capability is a new folder — no engine edit, ever.

`manifest.json` declares `"kind": "deterministic" | "prompt_backed"`, the MCP tool schema, and a `requires` list of external binaries (e.g. `["ruff", "git"]`) so the engine can fail loudly at startup if a dependency is missing rather than mid-run.

### The Findings model (shared currency)

Every plugin emits zero or more **Findings**. A Finding is the common language between any capability and the GitHub output layer — this is what lets you add capabilities without touching the output mechanics. A Finding:

```python
@dataclass
class Finding:
    id: str                 # stable, e.g. "secrets.history_key.aws_3f2a"
    severity: Severity      # CRITICAL | ADDITIVE | ADVISORY
    title: str              # issue title
    body: str               # markdown detail: what, why, how to fix
    suggested_action: str   # human-readable remediation
    artifact: str | None    # path to a generated file, if ADDITIVE
    labels: list[str]       # e.g. ["open-source", "security", "automated"]
```

Severity drives the output action (§4), and it is the *plugin's* job to set it honestly. The engine never reclassifies. A secrets plugin emits CRITICAL; a codemeta generator emits ADDITIVE with an artifact attached; a linter emits ADVISORY.

### Handler kinds — the deterministic / prompt-backed line

This line is non-negotiable and is encoded in each manifest:

- **Deterministic** — real code, same input → same output. Secrets scan, DOI resolution, license-as-fact, lint, doc coverage, schema validation. Use wherever correctness matters.
- **Prompt-backed** — loads `SKILL.md`, calls Claude via `ctx.claude()`, returns judgment-based drafts (README prose, license *rationale*, docstring-quality assessment). Non-deterministic; advisory or draft-only, never authoritative on fact.

**Never let a model decide whether a secret is present or whether a DOI resolves.**

### Transport
Start **stdio** — server runs as a local subprocess; no network, no auth, simplest to debug. Move to **SSE/HTTP** only when remote partner access is a concrete need, at which point auth and the prompt-injection threat (repo content flowing into a model is an injection vector — treat all repo content as untrusted) become first-order.

### State
MCP is stateless per conversation. The workflow is multi-step, so the *client* holds sequence across turns; the engine stays stateless within a run. For resumable migrations, persist a `os-migration-state.json` manifest. (Caveat carried from v1: this couples tool state to the artifact; abandoned migrations leave cruft. Acceptable, but known.)

---

## 3. The capability set

Marked **[D]** deterministic / **[P]** prompt-backed. Grouped by module. Each is an independent plugin; the list is a roadmap, not a build-all-at-once mandate.

### Safety module (gates everything)
| Plugin | Type | Function |
|---|---|---|
| `secrets_scan` | D | Scans working tree **and git history** for keys/credentials. Emits CRITICAL. Must run first and gate the workflow. |
| `pii_scan` | D | Flags SSNs, emails, other PII patterns in data/code. CRITICAL. |

### Legal & licensing module
| `license_check` | D | Detects existing license; analyzes dependency licenses for incompatibility (e.g. GPL contamination). Facts only. |
| `license_recommend` | P | Recommends an OSI license with rationale, honoring funder mandates (federal/NASA). Draft + rationale, human decides. |

### Academic integrity module (the differentiator)
| `doi_verify` | D | Resolves every DOI in references/CITATION via Crossref/DataCite REST API; flags dead or mismatched DOIs. |
| `citation_cff` | D+P | Generates valid `CITATION.cff` (schema deterministic; author/ORCID/grant extraction prompt-assisted). ADDITIVE. |
| `codemeta_gen` | D+P | Generates `codemeta.json` (research-software metadata standard). ADDITIVE. |
| `zenodo_ready` | D | Checks Zenodo-archival readiness; generates `.zenodo.json`. Does not mint (needs auth). ADDITIVE. |

### Reproducibility module
| `repro_check` | D | Verifies pinned deps, environment spec, data availability statement, runnable example against FAIR norms. |
| `env_capture` | D+P | Generates missing `environment.yml` / `Dockerfile` / reproduce script. ADDITIVE. |

### Quality module
| `lint_run` | D | Wraps `ruff`/`eslint`/`lintr` per detected language; aggregates violations. ADVISORY. |
| `doc_coverage` | D | Measures docstring/JSDoc coverage (`interrogate`-style). Reports a number. ADVISORY. |
| `doc_quality` | P | Assesses whether docstrings are useful vs. restating names. ADVISORY, low-stakes. |

### Documentation module
| `readme_gen` | P | Drafts research-tuned README: purpose, install, usage, **data provenance, methods/reproducibility, how to cite, funding, related publications, data availability**. ADDITIVE. |
| `contributing_gen` | P | Drafts CONTRIBUTING.md + code of conduct. ADDITIVE. |

### Orchestration & output
| `assess_readiness` | D | Runs the deterministic scan suite, aggregates a scorecard. |
| `github_issues` | D | Turns Findings into GitHub issues/PRs (see §4). |
| `migration_plan` | P | Synthesizes all findings into an ordered, human-reviewable checklist surfacing decision points. |

**Standards posture:** conform to existing research-software standards (CITATION.cff, codemeta, datapackage/Frictionless, Zenodo). Do **not** invent a "VERSO format." The README is a *section template* layered on the standard, not a new format.

---

## 4. The GitHub output layer — issues-first

**Default is to create issues, not to fix.** This is a hard stance, strongest in the academic context: the tool must never silently author changes that become part of the scholarly record. The human is the author of every change.

### Three-tier action model

The engine maps Finding severity to action. This mapping lives in the core, so every current and future plugin inherits it for free:

| Severity | Examples | Action |
|---|---|---|
| **CRITICAL / destructive** | Secret in git history, PII, license incompatibility | **Issue only.** Never touch the repo. History rewrites and credential purges are destructive and irreversible — the issue tells the human exactly what to run. |
| **ADDITIVE** | Missing CITATION.cff, codemeta.json, README, Dockerfile | **Pull request** with the generated file. Never a direct push to main. Human reviews and merges. |
| **ADVISORY** | Low doc coverage, lint violations, weak docstrings | **Issue** listing findings; optionally a PR for auto-fixable lint only. |

Auto-fix-via-PR is a **v2-of-this-project** capability. Lead with issues only; earn trust first. One bad autonomous edit kills adoption on campus.

### How `github_issues` works mechanically

```
Findings[]  ──►  github_issues plugin  ──►  GitHub REST API (or GitHub MCP server)
                       │
                       ├─ dedupe: stable Finding.id ↔ issue (search by hidden marker
                       │   comment <!-- skill-mcp:secrets.history_key.aws_3f2a -->)
                       │   so re-runs UPDATE existing issues, never duplicate
                       ├─ CRITICAL/ADVISORY → create or update issue, apply labels
                       │   ("automated", "open-source", severity, module)
                       ├─ ADDITIVE → create branch, commit artifact, open PR
                       │   body links back to the originating Finding
                       └─ summary issue ("Open-source readiness: 7 items") that
                           checklists and links all child issues/PRs
```

Idempotency is the key property: the stable `Finding.id` embedded as an HTML marker comment means running the tool weekly updates the same issues rather than spamming new ones. This is what makes it usable as an ongoing check, not a one-shot.

### Auth for writing to GitHub
Writing issues/PRs requires a token. Two supported paths:
- **Personal Access Token (fine-grained)** scoped to the target repo, passed via env var — simplest for a single PI's own repo.
- **GitHub App** installation — the right answer for VERSO running this across many partner repos; per-repo install, scoped permissions (issues: write, contents: write, pull_requests: write), revocable, auditable. Build for PAT first, design the auth interface so a GitHub App slots in without core changes.

---

## 5. Input & access — feeding it the repo

You're right: the tool takes **a local path to cloned code** as the primary input, plus **an optional repo URL** for context (owner/name needed to write issues back).

### The two inputs, decoupled on purpose
- **Local path** — required for all *analysis*. The engine never clones; analysis plugins read `ctx.repo_path`. This keeps analysis offline, fast, and testable, and means it works on code that isn't on GitHub at all.
- **Repo URL** — required only for the *output* layer (`github_issues`), to know where to write. Analysis works without it.

Decoupling these means the analysis modules have zero GitHub dependency — you can run the whole readiness assessment on a local folder with no network and no token. Only writing back needs GitHub.

### Private repositories
Analysis doesn't need GitHub access at all — it reads the local clone. So **the access question is really two separate questions:**

1. **Getting the code locally (read).** The user clones it themselves with their own credentials and hands the tool a path. The tool never needs the user's Git credentials. For a private repo, the human runs `git clone` (SSH key or their own token), then points the tool at the folder. This is the safest model — the tool never holds clone credentials. *Optionally*, a convenience `clone` helper can accept a token and clone for them, but the default and recommended path is "you clone, I analyze."
2. **Writing issues back (write).** This is where auth is unavoidable, and it's the PAT/GitHub App from §4 — a token scoped to that private repo with issues/PR write permission. A fine-grained PAT can be scoped to a single private repo, which is exactly what you want.

So for a **private repo**: human clones locally with their existing access → tool analyzes the folder offline → if writing issues, a fine-grained PAT (or GitHub App install) scoped to that repo authorizes the writes. The tool's standing access is minimal and the destructive-action surface is limited to issue/PR creation, never history rewrites.

---

## 6. What must be built or downloaded

### Built (this project)
- The **engine**: plugin loader, registry, dispatch, `WorkContext`, `Finding` model, severity→action mapping, MCP server wiring.
- The **GitHub output adapter** (issues/PRs, dedupe, summary issue).
- The **plugins** in §3, sequenced per §8.
- The **SKILL.md** files for prompt-backed plugins (where the open-source/research best-practice knowledge lives).
- The **fixture test corpus** (§7) — built, not downloaded; it's the deliverable that makes testing real.

### Downloaded / dependencies
- Python 3.10+, official `mcp` SDK, `anthropic` SDK.
- `gitpython` (repo/history inspection), `detect-secrets` or `trufflehog` (secrets — history-aware), `cffconvert` (CITATION.cff validation), `interrogate` (doc coverage).
- Linters per supported language: `ruff` (Python), `eslint` (TS/Svelte/Vue — ORCA stack), `lintr` (R).
- `PyGithub` or the GitHub MCP server (issue/PR writes).
- For DOI verification: nothing to install — Crossref/DataCite are public REST APIs over HTTPS.
- **MCP Inspector** (dev harness) and optionally **Claude Desktop** (to test as a real client).

---

## 7. Testing — the deliverable, not the afterthought

A tool that open-sources repos can ship a leaked credential to public GitHub. **The test suite is what stands between the team and that headline.** Modularity helps here: each plugin owns its own `tests/` and fixtures, so test coverage scales with capabilities and a broken plugin can't silently degrade the suite.

### Fixture repo corpus (build FIRST, before handlers)
Hand-crafted small Git repos under `tests/fixtures/`:
- Clean repo that should pass (LICENSE, README, tests, pinned deps).
- **Planted secret in working tree.**
- **Planted secret in git history, removed from working tree** — the case naive scanners miss.
- PII patterns in a data file.
- GPL dependency constraining the license recommendation.
- Repo with broken/dead DOIs and one DOI pointing to the wrong paper.
- Large binaries that shouldn't be committed.
- No license, no README, broken dependency manifest.
- Already-fully-open-source repo (idempotency / no-op check).
- Empty repo (edge case).

### Deterministic plugins — standard rigorous testing
pytest + the fixtures. The secrets scanner specifically: **a planted-secret corpus with zero tolerance for false negatives.** A miss is a build-breaking failure, not a warning. DOI verification: test against known-good, known-dead, and known-mismatched DOIs (can mock the Crossref response to keep CI offline and deterministic).

### Prompt-backed plugins — three layers
1. **Contract/schema tests (deterministic, in CI):** output must be *valid* even though content varies — CITATION.cff parses, license recommendation names a real OSI license, README contains required sections.
2. **LLM-as-judge evals:** run generators against fixtures, score output with a separate Claude call + rubric; track scores as SKILL.md files change (borrow the skill-creator eval scaffolding). 10–15 repo eval set.
3. **Human review on a holdout** for high-stakes outputs (license recommendation especially). No metric replaces this for legal judgment.

### Integration tests — the negative test is the most important
- **Plant a secret, run the full workflow, assert it HALTS** and refuses to emit a "ready to publish" plan. If the safety gate can be bypassed by orchestration, nothing else matters.
- Idempotency: run `github_issues` twice against a fixture; assert issues are *updated*, not duplicated (mock the GitHub API).
- Plugin isolation: assert a deliberately-broken plugin fails its own tool call without taking down the engine or other plugins.

### Acceptance test (Phase 6)
Run end-to-end on a real closed VERSO/ORCA repo; a human who knows research open-source best practices reviews the output package. Passes if they'd actually follow the plan and it surfaces decision points rather than silently guessing.

---

## 8. Build plan — modular, phased

Each phase ends in something demonstrable. Modularity means later phases add plugins without touching the core built in Phase 1.

**Phase 0 — Validate demand (1 wk, no code).** Interview 2–3 PIs. If "I'd just use a checklist" wins, build the checklist instead, or reframe explicitly as a teaching artifact. Proceed only if repeatable-tooling demand is real.

**Phase 1 — Engine + plugin contract (wk 1–2).** MCP server (stdio), plugin loader/registry, `WorkContext`, `Finding` model, severity→action mapping. One trivial deterministic plugin (`license_check`, LICENSE-file presence) end-to-end. **Exit:** Claude discovers and calls the plugin; a second dummy plugin proves drop-in works with zero engine edits.

**Phase 2 — Safety gate (wk 3).** `secrets_scan` (history-aware) + `pii_scan`, deterministic. Build the planted-secret fixture corpus first. **Exit:** zero false negatives on planted-secret set.

**Phase 3 — GitHub output layer (wk 4–5).** `github_issues`: Findings→issues, dedupe via marker comments, summary issue, PAT auth. Issues only — no PRs yet. **Exit:** run against a test repo, produce correctly-labeled deduplicated issues; second run updates rather than duplicates.

**Phase 4 — Academic integrity module (wk 6–7).** `doi_verify`, `citation_cff`, `codemeta_gen`, `zenodo_ready`. This is the differentiator — front-load it. ADDITIVE findings emit generated files (still delivered as issues w/ attached artifacts until PRs land). **Exit:** DOI verification catches planted bad DOIs; valid CITATION.cff + codemeta.json generated for a real repo.

**Phase 5 — Quality + reproducibility (wk 8–9).** `lint_run`, `doc_coverage`, `repro_check`, `env_capture`. Mostly wrapping existing tools. **Exit:** aggregated lint/coverage scorecard on a real ORCA repo.

**Phase 6 — Generators + orchestration + PRs (wk 10–11).** `readme_gen`, `contributing_gen`, `license_recommend`, `migration_plan`. Add ADDITIVE→PR path to `github_issues`. **Exit:** end-to-end run on a real closed repo produces a complete reviewable package; PRs open for generated files.

**Phase 7 — Dogfood + document (wk 12).** Open-source an actual closed VERSO/ORCA project with it. Write deployment docs + the plugin-authoring guide (how to add a capability without touching the core). Decide stdio vs. SSE. **Exit:** one real repo migrated; decision points surfaced, not auto-resolved.

### Tech stack
Python (ORCA competency + official SDK). `mcp`, `anthropic`, `gitpython`, `detect-secrets`/`trufflehog`, `cffconvert`, `interrogate`, `ruff`/`eslint`/`lintr`, `PyGithub`. Repo on `github.com/VERSO-UVM`, Apache-2.0 or MIT (dogfood the recommendation).

---

## 9. Deployment

**Phase 1–7 (development & single-PI use): stdio, local.** No hosting. The PI installs via `pip`/`uv`, registers the server in their MCP client config (Claude Desktop reads a JSON config that names the command to spawn), points it at a cloned folder. A `GITHUB_TOKEN` env var enables writes. Zero infrastructure. This is the recommended deployment for the foreseeable future.

**If/when remote multi-partner access is needed: SSE/HTTP.** This is a real step up in cost and risk, not a config flag:
- Authentication on the server (who may call it).
- The **prompt-injection threat becomes first-order** — repo content flowing into prompt-backed plugins is an injection vector; sandbox and treat all repo content as untrusted input.
- A **GitHub App** replaces PATs for scoped, auditable, revocable multi-repo access.
- Hosting, logging, secret management.

Don't pay this cost until a concrete partner need justifies it. Most of VERSO's value is capturable with the local stdio model.

---

## 10. Open questions for scoping
1. Is demand real, or is a static checklist enough? (Phase 0.)
2. Who owns maintenance as standards drift after the semester?
3. Local stdio is far simpler — is there a concrete partner need for remote/SSE that justifies the auth/security/hosting cost?
4. VERSO product or teaching artifact? Different quality bars.
5. PAT vs. GitHub App — when does the multi-repo case force the App?
6. Does the academic-integrity module alone justify the project even if the rest is deferred? (It might — it's the part nothing else does.)