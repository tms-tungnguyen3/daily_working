# Policy: Tracker Adapter Interface

This pipeline talks to exactly one task tracker per project, selected by `task_tracker` in `.claude/daily-working.yml`. [phases/fetch-task](../phases/fetch-task.md), [templates/tracker-write](../templates/tracker-write.md), and [phases/implement](../phases/implement.md)/[phases/close](../phases/close.md) never hardcode a specific tracker's mechanics — they call the contract below generically, and dispatch to `adapters/<task_tracker>.md` for the concrete "how." This is what keeps the skill a coordinator instead of a Redmine-only tool: adding a tracker means writing one new adapter file, not touching any phase.

## The contract

An adapter file (`adapters/<task_tracker>.md`) implements exactly these four operations:

### 1. `fetch(id)` → task record

Input: whatever form of ID the adapter accepts — numeric ID, ticket key, issue number, a pasted URL. The adapter's own "Resolve what to open" section says which forms it takes and how it turns them into something it can query.

Output, normalized to these fields regardless of tracker:

| Field | Meaning |
|---|---|
| `subject` | one-line title |
| `description` | the body/requirement text |
| `status` | the tracker's current status/state, in its own native vocabulary — **not** normalized to a shared enum; each adapter documents what its own values look like (e.g. Redmine's `Resolved`/`Closed`/`Rejected` vs. GitHub's `open`/`closed` + labels) so [phases/fetch-task](../phases/fetch-task.md)'s already-closed check can read it correctly |
| `assignee` | who it's currently assigned to, or empty |
| `comments[]` | full history in order, each with author/timestamp/body |
| `attachments[]` | each with a way to retrieve its content (a URL, a file, or — if the tracker has no real attachment concept — a note that images/logs only ever appear embedded in `description`/`comments`) |

An adapter may return extra fields beyond this table (e.g. Redmine's `custom_fields`) — phases read them opportunistically, but never *require* a field outside this table to function.

### 2. `write_comment(id, text)`

Post a plain-text comment. No status change.

### 3. `set_status(id, status_key)`

Transition to one of exactly two abstract keys this pipeline ever uses:

- **`in_progress`** — set once in [phases/implement](../phases/implement.md), before any code is written
- **`in_review`** — set once in [phases/close](../phases/close.md), after the PR is open and browser-verified — **never** a terminal/closed state; final close is a separate, later, human-triggered step (see [phases/close](../phases/close.md))

Each adapter maps these two keys to whatever concrete representation its tracker actually uses — a numeric status ID, a label, a project-board column, a workflow transition — via that adapter's own config block. A tracker with no native "status" concept at all (e.g. plain GitHub Issues) is free to implement this as a label add/remove instead of a real state transition; the contract only cares that calling it makes the tracker visibly reflect "someone is on this" / "this is ready for review."

## Adding a tracker

1. Pick a value for `task_tracker` and a config block name (e.g. `linear:`).
2. Write `adapters/<name>.md` implementing the four operations above against that tracker's actual API/CLI/UI, including its own "Resolve what to open" and attachment-handling sections.
3. Document the `in_progress`/`in_review` mapping inside that adapter's config block — [adapters/redmine](../adapters/redmine.md) and [adapters/github](../adapters/github.md) show two different shapes (numeric status IDs vs. labels) for the same two keys.
4. Nothing under `phases/`, `workflows/`, or `templates/tracker-write.md` needs to change.

## Today's adapters

- [adapters/redmine](../adapters/redmine.md) — via the already-authenticated `claude-in-chrome` browser session, no credentials stored
- [adapters/github](../adapters/github.md) — via the `gh` CLI, reusing the same auth [templates/pr](../templates/pr.md) already depends on
