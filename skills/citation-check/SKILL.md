---
name: citation-check
description: Check citation integrity in an academic repository. Verifies that DOIs resolve and match what they claim, validates or generates CITATION.cff and codemeta.json, and checks Zenodo archival readiness. This is the academic differentiator — use it before open-sourcing any research repository.
license: MIT
compatibility: Requires git, curl, and python3; cffconvert recommended for CITATION.cff validation. The path argument replaces $ARGUMENTS (default: current directory).
metadata:
  author: VERSO-UVM
  version: "1.0"
argument-hint: "[path-to-repo]"
context: fork
agent: claude
allowed-tools: Read Bash(find *) Bash(grep *) Bash(curl *) Bash(python *) Bash(cffconvert *)
---

Check the citation integrity and academic metadata of the repository at `$ARGUMENTS` (default: current working directory).

## Step 1 — Find all DOIs in the repository

Search for DOI references across README, CITATION.cff, codemeta.json, papers, and code comments:

```bash
grep -rn --include="*.md" --include="*.rst" --include="*.txt" --include="*.cff" \
  --include="*.json" --include="*.bib" --include="*.tex" \
  -oE "https?://doi\.org/[^ )\"'\`]+" $REPO_PATH | sort -u

grep -rn --include="*.md" --include="*.cff" --include="*.bib" \
  -oE "doi:\s*[0-9]{2}\.[0-9]{4,}/[^ \"'\`]+" $REPO_PATH | sort -u
```

### Step 2 — Verify each DOI resolves

For each DOI found, verify it resolves via the Crossref REST API:

```bash
curl -sI "https://doi.org/THE_DOI" | grep -E "^(HTTP|Location)"
```

Or use the Crossref metadata API for richer information:
```bash
curl -s "https://api.crossref.org/works/THE_DOI" | python3 -c "
import sys, json
data = json.load(sys.stdin)
if data.get('status') == 'ok':
    msg = data['message']
    print('Title:', msg.get('title', ['?'])[0])
    print('Type:', msg.get('type', '?'))
    print('Year:', msg.get('published', {}).get('date-parts', [['']])[0][0])
else:
    print('NOT FOUND')
"
```

Report:
- ✅ Resolves and metadata matches context
- ⚠️ Resolves but title/year doesn't match what the repo claims
- ❌ DOI does not resolve (dead link)

### Step 3 — Validate CITATION.cff

Check whether `CITATION.cff` exists:
```bash
ls $REPO_PATH/CITATION.cff 2>/dev/null || echo "MISSING"
```

If it exists, validate it:
```bash
# If cffconvert is installed
cffconvert --validate -i $REPO_PATH/CITATION.cff 2>&1

# Otherwise, check it's valid YAML with required fields
python3 -c "
import yaml, sys
with open('$REPO_PATH/CITATION.cff') as f:
    data = yaml.safe_load(f)
required = ['cff-version', 'message', 'authors', 'title']
for field in required:
    if field not in data:
        print(f'MISSING required field: {field}')
    else:
        print(f'  {field}: present')
print('type:', data.get('type', '(missing)'))
"
```

Check for:
- `cff-version: 1.2.0` (current version)
- `message` field
- `authors` with `family-names`, `given-names` (and optionally `orcid`)
- `title`
- `version` and `date-released` for software
- `doi` or `repository-code`

### Step 4 — Check codemeta.json

```bash
ls $REPO_PATH/codemeta.json 2>/dev/null || echo "MISSING"
```

If it exists, validate key fields:
```bash
python3 -c "
import json
with open('$REPO_PATH/codemeta.json') as f:
    data = json.load(f)
for field in ['@context', '@type', 'name', 'author', 'license', 'version']:
    print(field + ':', 'present' if field in data else 'MISSING')
"
```

### Step 5 — Zenodo readiness check

Check `.zenodo.json` if present:
```bash
ls $REPO_PATH/.zenodo.json 2>/dev/null || echo "No .zenodo.json — optional but recommended for DOI minting"
```

Check for a GitHub release tag (needed for Zenodo auto-archival):
```bash
git -C $REPO_PATH tag --list | tail -5
```

### Step 6 — Generate missing files (draft only)

If `CITATION.cff` is missing, draft one based on what you can infer from the repository:
- Project name from README or directory name
- Authors from `git log --format='%aN <%aE>' | sort -u | head -10`
- Language/type from files present
- Any existing DOI, URL, or version tag

Mark the draft clearly: `# DRAFT — review and fill in bracketed fields before committing`

If `codemeta.json` is missing, generate a draft similarly.

## Output format

```
## Citation Check: [repo name]

### DOI Verification
| DOI | Status | Notes |
|-----|--------|-------|
| https://doi.org/... | ✅ Resolves | Title matches |
| https://doi.org/... | ❌ Dead | 404 from doi.org |
| https://doi.org/... | ⚠️ Mismatch | Resolves to different paper |

### CITATION.cff
- Status: ✅ present and valid / ⚠️ present but invalid / ❌ missing
- [validation errors or missing fields if any]

### codemeta.json
- Status: ✅ / ⚠️ / ❌
- [missing fields if any]

### Zenodo Readiness
- .zenodo.json: present / missing (optional)
- Release tags: [list or "none"]

### Generated drafts
[If any files were drafted, list them here and note they need human review]

### Summary and recommendations
[ordered list of actions]
```
