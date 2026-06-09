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
