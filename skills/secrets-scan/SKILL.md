---
name: secrets-scan
description: Scan a repository for hardcoded secrets, API keys, credentials, and PII in both the working tree and git history. This is the safety gate — always run before open-sourcing any academic repository. Emits CRITICAL findings for anything that would be exposed on a public GitHub repo.
argument-hint: "[path-to-repo]"
context: fork
agent: claude
allowed-tools: Read Bash(find *) Bash(grep *) Bash(git log *) Bash(git show *) Bash(git diff *) Bash(git rev-list *) Bash(trufflehog *) Bash(detect-secrets *)
---

Scan the repository at `$ARGUMENTS` (default: current working directory) for secrets, credentials, and PII that would be exposed if the repo were made public.

**This is a CRITICAL safety gate. If any finding is emitted, report it clearly and stop — do not produce any "ready to publish" assessment until the researcher has resolved it.**

## Scan approach

### Step 1 — Use a dedicated secrets scanner if available

Check whether `trufflehog` or `detect-secrets` is installed:

```bash
which trufflehog || which detect-secrets
```

If `trufflehog` is available, run it on the full git history (the most important case — secrets removed from working tree but still in history):
```bash
trufflehog git file://$REPO_PATH --only-verified 2>/dev/null
```

If `detect-secrets` is available:
```bash
detect-secrets scan $REPO_PATH
```

If neither is available, proceed with the manual grep approach below and note the limitation.

### Step 2 — Manual grep scan (working tree)

Scan the working tree for high-signal patterns. Skip `.git/`, `node_modules/`, `__pycache__/`, `venv/`, `.venv/`.

Key patterns to check:
```bash
grep -rn --include="*.py" --include="*.js" --include="*.ts" --include="*.env" \
  --include="*.yaml" --include="*.yml" --include="*.json" --include="*.toml" \
  --include="*.cfg" --include="*.ini" --include="*.rb" --include="*.sh" \
  -E "(password|passwd|secret|api_key|apikey|token|private_key|access_key)\s*[=:]\s*['\"][^'\"]{8,}" \
  $REPO_PATH
```

Check for specific high-value credential patterns:
- AWS: `AKIA[0-9A-Z]{16}`
- GitHub tokens: `ghp_[a-zA-Z0-9]{36}` or `github_pat_`
- OpenAI/Anthropic: `sk-[a-zA-Z0-9]{48}` or `sk-ant-`
- Private keys: `-----BEGIN (RSA|EC|OPENSSH|PGP) PRIVATE KEY`
- Generic high-entropy strings in assignment context

### Step 3 — Git history scan

This is the case that naive scanners miss: secrets committed, then deleted.

```bash
# List all files ever committed (including deleted)
git -C $REPO_PATH log --all --full-history --name-only --format="" | sort -u | grep -E "\.(env|key|pem|p12|pfx|cert|secret)$"

# Search patches for credential patterns
git -C $REPO_PATH log -p --all -S "AKIA" --source --all 2>/dev/null | head -100
git -C $REPO_PATH log -p --all -S "ghp_" --source --all 2>/dev/null | head -100
git -C $REPO_PATH log -p --all -S "sk-" --source --all 2>/dev/null | head -100
git -C $REPO_PATH log -p --all --pickaxe-regex -S "password\s*=\s*['\"][^'\"]{8,}" 2>/dev/null | head -100
```

### Step 4 — PII scan

Look for personally identifiable information in data files:

```bash
# SSN pattern
grep -rn --include="*.csv" --include="*.tsv" --include="*.json" --include="*.txt" \
  -E "\b[0-9]{3}-[0-9]{2}-[0-9]{4}\b" $REPO_PATH

# Email addresses in bulk (>5 on a single line or in a list file — not code)
grep -rn --include="*.csv" --include="*.tsv" --include="*.txt" \
  -cE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" $REPO_PATH | \
  awk -F: '$2 > 5'
```

## Output format

```
## Secrets & PII Scan: [repo name]

### Tool used
[trufflehog / detect-secrets / manual grep — note any gaps]

### Findings

#### 🔴 CRITICAL — [finding title]
- **Location:** [file:line or commit SHA]
- **Pattern matched:** [what was found, redacted — show type not value]
- **In history only (removed from working tree):** yes/no
- **Remediation:** [exact steps — e.g., git filter-repo command, BFG command, or credential rotation URL]

[repeat for each finding]

### Summary
- CRITICAL findings: N
- [If 0]: No secrets or PII detected. ✅ Safe to proceed with open-sourcing (this scan does not guarantee completeness — a dedicated scanner like trufflehog is recommended for final verification).
```

## Important notes

- **Never print the actual secret value** — show the type/pattern (e.g., "AWS access key") and location only.
- **History secrets are as dangerous as working-tree secrets.** A secret removed in a later commit is still accessible via `git log` on a public repo.
- **Remediation for history secrets** requires `git filter-repo` (preferred) or BFG Repo Cleaner, and a force push. Tell the researcher this explicitly — it is destructive and requires coordination with collaborators.
- **Always recommend rotating the credential**, even if it's been removed from history — treat it as compromised.
