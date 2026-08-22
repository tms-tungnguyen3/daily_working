# daily-working

A Claude Code plugin/skill that coordinates a full task-to-verified-change loop:

1. **Fetch** the task/ticket from Redmine by ID.
2. **Implement** the change with the Claude CLI, following the target project's existing conventions.
3. **Verify** the change in a real running UI via the `claude-in-chrome` extension.

See [`skills/daily-working/SKILL.md`](skills/daily-working/SKILL.md) for the full workflow this skill runs.

## Install

**As a plugin (recommended once published):**

```bash
/plugin marketplace add <owner>/<repo>
/plugin install daily-working@daily-working-marketplace
```

*(Replace `<owner>/<repo>` with wherever this repo ends up hosted, e.g. on GitHub, once pushed.)*

**As a personal skill** (available in every project, no plugin needed):

```bash
mkdir -p ~/.claude/skills/daily-working
cp skills/daily-working/SKILL.md ~/.claude/skills/daily-working/SKILL.md
```

**As a project skill** (checked into one specific repo, shared with anyone using it):

```bash
mkdir -p <target-project>/.claude/skills/daily-working
cp skills/daily-working/SKILL.md <target-project>/.claude/skills/daily-working/SKILL.md
```

## Requirements

- The [`claude-in-chrome`](https://code.claude.com/docs/en/claude-in-chrome) browser extension, for Phase 1 (optional) and Phase 3 (verification).
- A Redmine instance with either an authenticated browser session or an API key (`REDMINE_URL` / `REDMINE_API_KEY`).
- Project-specific conventions (test framework, DB safety rules, commit format) are assumed to already exist in the target codebase — this skill coordinates around them, it doesn't define them.

## License

MIT — see [LICENSE](LICENSE).
