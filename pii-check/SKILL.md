---
name: pii-check
description: Audit a repo for PII leakage, secrets, and sensitive data before commits or publishing. Runs gitleaks if available, scans git history, checks working tree for known and novel patterns, and optionally wires up a pre-commit hook. Use before making a private repo public, after importing external data into a project, or as part of a pre-release checklist.
---

# PII & Secrets Check

Audit the current repo for personally identifiable information, credentials, and sensitive data. Covers static patterns, novel inference, git history, and optionally installs a prevention hook.

## Installation

```bash
# Clone the skills repo
git clone https://github.com/clewisdev/skills.git

# Copy this skill into your Claude Code skills directory
cp -r skills/pii-check ~/.claude/skills/pii-check
```

Then use `/pii-check` in any Claude Code session, or trigger it naturally (see Activation below).

## Activation

User says: "pii check", "scan for pii", "check for secrets", "pre-publish audit", /pii-check.
Also trigger when: user is about to `git push` a previously private repo, or asks "is it safe to make this public?".

## Prerequisites — install git-filter-repo if needed

Before running any step that rewrites history, check whether `git-filter-repo` is available:

```bash
# Pinned to v2.47.0 — update VERSION and EXPECTED_SHA together when upgrading
VERSION="v2.47.0"
EXPECTED_SHA="67447413e273fc76809289111748870b6f6072f08b17efe94863a92d810b7d94"
DEST="/tmp/git-filter-repo"

if ! python3 "$DEST" --version 2>/dev/null; then
  curl -sL "https://raw.githubusercontent.com/newren/git-filter-repo/${VERSION}/git-filter-repo" -o "$DEST"
  ACTUAL_SHA=$(sha256sum "$DEST" | awk '{print $1}')
  if [ "$ACTUAL_SHA" != "$EXPECTED_SHA" ]; then
    echo "ABORT: git-filter-repo checksum mismatch (got $ACTUAL_SHA)" >&2
    rm -f "$DEST"
    exit 1
  fi
  chmod +x "$DEST"
fi
```

Use `python3 /tmp/git-filter-repo` for all subsequent calls — do NOT try `apt`, `pip`, or `pip3` first; they require sudo or a venv and fail silently in WSL. The curl approach is always reliable.

> After running `git filter-repo`, it removes the `origin` remote automatically. Re-add it before pushing:
> ```bash
> git remote add origin <url>
> git push origin main --force
> ```

## Steps (run in order)

### 1. Run gitleaks (if available)

```bash
which gitleaks && gitleaks detect --source . --no-git 2>/dev/null
which gitleaks && gitleaks detect --source . 2>/dev/null   # includes git history
```

If gitleaks is not installed, tell the user:
> gitleaks is not installed. For full coverage, install it:
> - **Mac**: `brew install gitleaks`
> - **Windows (native)**: `winget install gitleaks`
> - **Linux / WSL**: download the pre-built binary:
>   ```bash
>   VER=$(curl -s https://api.github.com/repos/gitleaks/gitleaks/releases/latest | grep '"tag_name"' | cut -d'"' -f4 | tr -d v)
>   curl -sSL "https://github.com/gitleaks/gitleaks/releases/download/v${VER}/gitleaks_${VER}_linux_x64.tar.gz" \
>     | tar -xz -C /tmp gitleaks && sudo mv /tmp/gitleaks /usr/local/bin/gitleaks
>   ```
> Continuing with manual scan — install it before going public.

Parse gitleaks output; include any findings in the report. Do not stop — continue with manual checks below regardless.

### 2. Scan git history for sensitive content

```bash
git log --all --oneline           # check commit messages for names, emails, paths
git log --all -p -- '*.env' '*.json' '*.csv' '*.key' '*.pem' 2>/dev/null | head -200
git log --all --diff-filter=D --summary 2>/dev/null | grep -E '\.(env|pem|key|p12|pfx|csv)$'
```

Look for:
- Deleted files that once contained credentials or PII (deleted ≠ gone from history)
- Commit messages that mention real names, email addresses, or account identifiers
- Any commit that added then removed a credential (the exposure window still exists in history)

If any found: **flag as high severity** — history must be rewritten with `git filter-repo` before the repo is safe to publish.

> ⚠ **`git filter-repo` resets the working tree** to match the rewritten HEAD. Any local-only files that were previously tracked (even if now gitignored) will be deleted from disk. Before running:
> 1. Identify which files will be stripped: `git log --all --oneline -- <file>`
> 2. If those files exist locally and you want to keep them, copy their content somewhere safe first.
> 3. After filter-repo completes, restore them with the Write tool — they will be gitignored and won't be re-committed.
>
> The same wipe can happen during `git checkout` when switching to a branch where those files were tracked-then-deleted. Same precaution applies.

### 3. Scan working tree — known patterns

Use `grep -rn` (excluding `.git/`, `node_modules/`, `__pycache__/`, `venv/`, `*.pyc`):

| Category | Patterns to grep |
|----------|-----------------|
| Absolute paths | `/Users/<name>/`, `/home/<name>/`, `C:\Users\`, `/mnt/c/Users/` |
| Email addresses | `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}` — flag any that look real (not `you@example.com`, `user@gmail.com` placeholder patterns) |
| Partial masks | `[a-z]\*+@`, `[a-z]\*+[0-9]` — masked emails are worse than nothing, still identify the person |
| Phone numbers | `\b(\+?1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b` |
| API keys / tokens | `sk-`, `ghp_`, `ghs_`, `xox[baprs]-`, `AIza`, `AKIA`, `Bearer [A-Za-z0-9+/]{20,}` |
| App passwords | 4-word groups like `xxxx xxxx xxxx xxxx` — flag if not clearly a placeholder |
| Credentials in URLs | `://[^:]+:[^@]+@` (basic auth embedded in URLs) |
| Private keys | `BEGIN (RSA|EC|OPENSSH|PGP) PRIVATE KEY` |
| Social security / NI | `\b\d{3}-\d{2}-\d{4}\b`, `\b[A-Z]{2}\d{6}[A-Z]\b` |
| Real names in non-code | grep comments, markdown, CSV headers, and config files for names that appear in the git config (`git config user.name`) |

### 4. Novel / inference-based scan

Do not limit the scan to the above patterns. Also reason about:

- **Combination risk**: Fields that are benign alone (first name + postcode + employer) but identify a person together — flag if a file contains 3+ such fields about a real individual
- **Sample data that isn't synthetic**: CSVs, JSON fixtures, SQL seeds — if rows look like real people (plausible names, real-format phone numbers, non-example emails) rather than generated data, flag them
- **Internal URLs**: internal hostnames, VPN addresses, corporate SSO URLs — these reveal infrastructure even if no credential is present
- **Account identifiers**: usernames, customer IDs, subscription IDs tied to a real account — especially in test fixtures or config files
- **Comments with context**: inline comments that say "my account", "use Alice's key", "Bob's server" — flag as likely PII even if the value is redacted

### 5. Check for personal-context files in the index or history

These files are often created during development and contain session notes, account identifiers, or run details that don't belong in a public repo. gitleaks won't flag them because they contain no credential *patterns* — but they're still personal.

**Step 5a — check if any are currently tracked:**

```bash
git ls-files | grep -iE '(handoff|run[_-]history|session[_-]notes?|scratch|diary|notes?\.md|todo\.md|\.claude/)' 
```

Flag any matches as HIGH — they will be visible to anyone who clones the repo.

**Step 5b — check if any were ever in git history (even if now deleted or gitignored):**

```bash
git log --all --oneline -- handoff.md run_history.md session*.md notes.md scratch.md diary.md
git log --all --diff-filter=D --summary 2>/dev/null | grep -iE '(handoff|run.history|session.notes?|scratch|diary)'
```

Flag any matches as HIGH — `git rm --cached` and `.gitignore` do not remove files from history. History must be rewritten with `git filter-repo` before the repo is safe to publish.

**Step 5c — check .gitignore coverage for standard secrets:**

Verify these are in `.gitignore` (or equivalent):

```
.env
*.env.*
*.pem
*.key
*.p12
*.pfx
output/          # or whatever contains extracted data for this project
credentials.json
token.json
secrets/
```

Report any that are missing.

**Lesson from practice:** adding a file to `.gitignore` and running `git rm --cached` stops *future* tracking but leaves all prior commits intact. If a personal-context file was ever committed — even briefly — treat it as a history rewrite job before publishing.

### 6. Optionally install pre-commit hook

If gitleaks is installed and the user confirms, install a hook so future commits are checked automatically:

```bash
# .git/hooks/pre-commit
#!/bin/sh
gitleaks protect --staged -v
```

```bash
chmod +x .git/hooks/pre-commit
```

Tell the user: "This runs gitleaks on staged files before every commit. It does not catch secrets already in history — run `/pii-check` before publishing a repo regardless."

Note: this hook is local only (`.git/hooks/` is not committed). To share it with collaborators, use a tool like `pre-commit` (https://pre-commit.com) with the `gitleaks` hook in `.pre-commit-config.yaml`.

## Output format

Report findings grouped by severity. Be specific — include file path and line number for every finding.

```
## PII / Secrets Audit — [repo name] — [date]

### CRITICAL (block publish)
- git history contains deleted .env at abc1234 — rewrite history before publishing
- [file:line] API key pattern matched: sk-...

### HIGH (fix before publish)
- [file:line] Hardcoded absolute path: /mnt/c/Users/<username>/...
- [file:line] Real email address in non-example context

### MEDIUM (review)
- [file:line] Partial mask found — remove entirely or replace with placeholder
- .gitignore missing: output/

### LOW / INFO
- gitleaks not installed — manual scan only
- pre-commit hook not installed

### Clean
- No phone numbers found
- No private keys found
- ...

### Recommended next steps
1. ...
```

If nothing is found: say so explicitly — "No findings. Safe to publish." — but still note if gitleaks was absent.

## Severity rules

| Finding | Severity |
|---------|----------|
| Secret/credential in git history | CRITICAL |
| Live credential in working tree | CRITICAL |
| Real PII (email, name, path) in tracked file | HIGH |
| Partial mask | HIGH |
| Personal-context file in git history (handoff, run log, notes) | HIGH |
| Personal-context file currently tracked in index | HIGH |
| Missing .gitignore entry for sensitive dirs | MEDIUM |
| gitleaks not installed | LOW |
| pre-commit hook not installed | LOW |
