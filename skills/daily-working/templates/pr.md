# Template: Pull Request

Used by [phases/close](../phases/close.md). Open with `pr.method` from `.claude/daily-working.yml` (a companion skill/command, or plain `gh pr create` if unset — on a project already configured for [adapters/github](../adapters/github.md), this is the same `gh` auth that adapter uses).

Build title and body from what's already known instead of leaving it generic — a reviewer should be able to judge the change from the PR body alone, without having to go open the ticket first. The link is for traceability, not because the body should be thin.

**Title**: `{commit subject}` (or `[{ticket_key}] {subject}` if `commit.format` uses a ticket key) — searchable and consistent with the commit.

**Body**, when `pr.include_tracker_link` isn't explicitly `false`:

```markdown
## Task
<link to the ticket, from whichever adapter is active> — <subject>

## Summary
{what changed and why, 1-3 bullets}

## Tests
{pass/fail, coverage — from templates/summary.md}

## Browser verification
{what was exercised in phases/verify.md, what was observed}
```

The ticket link itself is adapter-specific — `<redmine.url>/issues/<id>` for [adapters/redmine](../adapters/redmine.md), the issue's `url` field (already returned by `fetch()`) for [adapters/github](../adapters/github.md). Use whatever the active adapter's `fetch()` gave back rather than reconstructing it.

No AI attribution: the body ends after Browser verification — no `Co-Authored-By: Claude` trailer, no `🤖 Generated with Claude Code` footer, no session link, in either the title or the body. Same rule as [policies/git](../policies/git.md) for the commit.
