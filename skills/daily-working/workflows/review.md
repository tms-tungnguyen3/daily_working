# Workflow: Review

Use this instead of [workflows/implement](./implement.md) when the code for a ticket **already exists** — the user (or someone else) already wrote the change, and wants it checked against the ticket rather than implemented from scratch. Typical triggers: "verify #4626 against the running app", "review this branch/PR against the ticket", "does this actually fix what the ticket asked for?".

Skip the branch/implement/database-safety steps entirely — nothing gets written here except (optionally) a comment on the task tracker.

## Sequence

1. **[phases/fetch-task](../phases/fetch-task.md)** — fetch the ticket for requirement context (read-only: skip the assignee/status "start work" gate, since no new work is starting). Still apply the ambiguity check — you can't verify against a requirement that doesn't clearly say what "correct" looks like.
2. **[phases/verify](../phases/verify.md)** — drive the real app in a browser and check the existing change against the ticket's requirement (and attachments, if any).
   - Matches → continue to step 3.
   - Doesn't match, or something's broken → report the gap to the user. Don't silently switch into [workflows/implement](./implement.md) to fix it — ask first, since that changes the scope of what was requested.
3. **[phases/close](../phases/close.md)** — report the [templates/summary](../templates/summary.md). If a PR doesn't exist yet, offer to open one per [templates/pr](../templates/pr.md); if it does, just note the verification result. The tracker close-out still needs the user's confirmation per [policies/safety](../policies/safety.md).

This workflow never runs [phases/setup](../phases/setup.md) on its own, but still needs `.claude/daily-working.yml` to exist (or tracker credentials to be available directly) for [phases/fetch-task](../phases/fetch-task.md) — run [workflows/implement](./implement.md) once first if this is genuinely the first time in this repo.
