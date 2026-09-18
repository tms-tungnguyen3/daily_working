# Phase 2: Implement via Claude CLI

Part of [workflows/implement](../workflows/implement.md). Runs after [phases/assess](./assess.md) has cleared the task as safe to proceed (low impact, sanity-checked).

## Create a branch (before writing any code)

See [policies/git](../policies/git.md) for the full rule and naming format:

```bash
git checkout -b <branch-name>
```

If another task from this skill is already in flight and uncommitted, don't check out a second branch in this same directory — see [policies/parallel](../policies/parallel.md) and give this task its own git worktree instead.

## Mark the ticket "In Progress" (once, before writing code)

Right now the task tracker only hears from this skill once — at [phases/close](./close.md), after everything is already done. For the whole span of this phase and [phases/verify](./verify.md), anyone looking at the ticket has no way to tell it's actively being worked. Fix that symmetrically with the [phases/close](./close.md) close-out: as soon as [phases/fetch-task](./fetch-task.md) ambiguity is resolved, call the active adapter's `set_status(id, "in_progress")` per [templates/tracker-write](../templates/tracker-write.md) — see the adapter file (e.g. [adapters/redmine](../adapters/redmine.md), [adapters/github](../adapters/github.md)) for what that abstract key actually does on this tracker (a status transition, a label, etc.).

- Only do this once ambiguity is resolved — don't mark something "in progress" while still waiting on a clarification comment.
- If the config doesn't yet record what `in_progress` maps to for this adapter (e.g. Redmine's `in_progress_status_id`, GitHub's `in_progress_label`), ask once and note it in `.claude/daily-working.yml` so future runs don't re-ask.
- If the project doesn't want automatic status transitions at all (some teams manage board state manually), a no-op `write_comment` ("Starting work on this.") is an acceptable lighter-weight fallback — ask which the user prefers the first time this comes up, same spirit as [phases/setup](./setup.md)'s questionnaire.
- Skip silently (don't block this phase) if neither a status convention nor comment preference is known and the user isn't available to ask right now — this step is a nice-to-have visibility signal, not a gate on doing the actual work.

## Implement

1. Follow the conventions recorded in `conventions.notes` (config file) — or this project's existing conventions if that field is empty — match the surrounding code rather than introducing a new pattern
2. Write/adjust tests — see [phases/test](./test.md) for the test conventions and the database-safety rule that governs every spec run
3. Commit using the format in [policies/git](../policies/git.md), built from the tracker's `subject` field

Do not mark the task done yet — implementation is only verified by tests until [phases/verify](./verify.md) confirms it in a real browser.
