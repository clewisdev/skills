# skills

A collection of Claude Code skills for use with the [Claude Code](https://claude.ai/code) CLI.

## What are skills?

Skills are markdown-based instruction files that extend Claude Code's behavior for specific tasks. Drop them into `~/.claude/skills/<skill-name>/SKILL.md` and they become available as slash commands (e.g. `/pii-check`).

Inspired by [anthropics/skills](https://github.com/anthropics/skills) and [mattpocock/skills](https://github.com/mattpocock/skills).

If you want a universal skills manager that works across AI coding tools beyond just Claude Code, see [airskills](https://github.com/chrismdp/airskills) by Chris Parsons and [asm](https://github.com/luongnv89/asm) by Luong Nguyen.

## Available skills

| Skill | Description |
|-------|-------------|
| [pii-check](./pii-check/SKILL.md) | Audit a repo for PII leakage, secrets, and sensitive data before commits or publishing. Runs gitleaks if available, scans git history, checks working tree for known and novel patterns, and optionally wires up a pre-commit hook. |

---

### pii-check — what makes it different

Most secrets scanners stop at pattern matching on the current working tree. `pii-check` adds three layers that nothing else covers together:

**1. Git history awareness.** Deleted files are not gone — they live in every clone's history until history is rewritten. This skill scans `git log --all` for files that were ever committed and then deleted, flags them as CRITICAL, and walks you through a `git filter-repo` rewrite (with a working-tree-wipe warning that most tutorials omit).

**2. Inference-based scanning.** Beyond grep patterns, it reasons about combination risk (first name + postcode + employer = identified person), flags sample data that looks real rather than synthetic, and catches internal hostnames and account identifiers that have no credential pattern but still expose infrastructure.

**3. Agentic-workflow artifacts.** It explicitly checks for `handoff.md`, session notes, run logs, and `.claude/` directories — files that are created by AI coding workflows and contain session context, account identifiers, or run details that gitleaks will never flag because they contain no credential *pattern*. These barely existed as a threat surface two years ago, which is why no traditional tool covers them.

### Closest neighbours

| Tool | Layer | Where it differs from pii-check |
|------|-------|----------------------------------|
| [sensitive-canary](https://github.com/coo-quack/sensitive-canary) | Context-window / API egress | Blocks secrets from reaching the Anthropic API via hooks — protects the *prompt*, not the repo. Complementary, not competing. |
| [vibe-guardian](https://github.com/Camof1ow/vibe-guardian) | Pre-commit hook | Broader vuln focus (injection, XSS, secrets), shallower on PII — regex/checklist only, no history awareness. |
| [scanning-for-secrets](https://github.com/jeremylongshore/claude-code-plugins-plus-skills) (jeremylongshore) | Pre-commit / audit | Secrets only via pattern matching and entropy. No PII, no history, no agentic-artifact checks. |
| [Anthropic's security-review skill](https://github.com/anthropics/skills) | Diff review | General security review scoped to the current diff. Not publish-readiness, not history. |

The gap: if you are about to `git push --mirror` a repo that started as a private workspace and has had an AI agent working in it, none of the above tools give you confidence it is safe to publish. `pii-check` does.

## Installation

Copy the skill folder into your Claude Code skills directory:

```bash
# Clone the repo
git clone https://github.com/clewisdev/skills.git

# Copy a skill
cp -r skills/pii-check ~/.claude/skills/pii-check
```

Then use it with `/pii-check` in any Claude Code session.

## Usage

| Trigger | Description |
|---------|-------------|
| `/pii-check` | Run a full PII and secrets audit on the current repo |
| `pii check` | Natural language trigger |
| `scan for pii` | Natural language trigger |
| `check for secrets` | Natural language trigger |
| `is it safe to make this public?` | Natural language trigger |

## Contributing

Pull requests welcome. Each skill lives in its own subdirectory with a single `SKILL.md` file.
