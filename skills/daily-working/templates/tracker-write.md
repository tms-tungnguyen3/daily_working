# Template: Tracker Writes

Every phase that needs to write to the tracker (post a comment, change status) calls the active adapter's `write_comment`/`set_status` per [policies/tracker-adapter](../policies/tracker-adapter.md) — the concrete mechanism lives in `adapters/<task_tracker>.md` ([adapters/redmine](../adapters/redmine.md), [adapters/github](../adapters/github.md)), not here. This file only holds the *what to say*, which is the same regardless of which tracker is active.

## Message templates by use case

### Ambiguity question ([phases/fetch-task](../phases/fetch-task.md))

`write_comment` — state the conflict specifically, don't ask vaguely:

> Should X behave as A or B when Y? Description says A, comment #3 implies B.

### Impact / sanity-check concern ([phases/assess](../phases/assess.md))

`write_comment` — state what's risky or what doesn't add up, and offer the narrower alternative if there is one:

> This ticket asks to change the default payment currency — that's a shared setting affecting every user, not just this account. Confirm you want that, or is the intent narrower?

> The bug this describes looks already fixed by commit `<sha>` — should I still proceed, or should the ticket be closed instead?

### In-progress marker ([phases/implement](../phases/implement.md))

`set_status(id, "in_progress")`, or a plain `write_comment` if the project prefers that lighter-weight signal — the active adapter's config says which:

> Starting work on this.

### Close-out comment ([phases/close](../phases/close.md))

`write_comment` — summarize what actually happened, don't just say "done":

> Implemented in `<commit sha/message>`. Verified in browser: `<what was exercised, what was observed>`. PR: `<PR link>`.

Followed by `set_status(id, "in_review")` — **never** a closed/terminal state at this point (see [phases/close](../phases/close.md) for why).
