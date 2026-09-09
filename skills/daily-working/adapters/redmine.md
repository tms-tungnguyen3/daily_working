# Adapter: Redmine

Implements the [policies/tracker-adapter](../policies/tracker-adapter.md) contract via the already-authenticated `claude-in-chrome` browser session. No API key involved anywhere — see [policies/browser](../policies/browser.md) for the tool-loading pattern this adapter uses for every operation below.

## Config block

```yaml
redmine:
  url:                     # or omit if using the REDMINE_URL env var; only used to build issue links, no API key involved
  ticket_key_prefix:       # e.g. "PROJ", or omit if tickets are plain numeric IDs
  user:                     # Redmine login/username this pipeline runs as — used for the assignee check
  in_progress_status_id:   # in_progress mapping: status_id to set when work starts, or omit to just write_comment instead
  in_progress_mode:        # "status" | "comment" | "skip" — how set_status("in_progress") behaves
  review_status_id:        # in_review mapping: status_id to set once implementation is browser-verified and a PR is open (e.g. "Review"/"Feedback") — NOT Resolved/Closed
```

## Resolve what to open

- User pasted a full issue URL → use it directly.
- User gave a bare ID (`4623`) or a ticket key seen in commits (uses `ticket_key_prefix`, e.g. `PROJ-4626` → strip the prefix to get the numeric issue ID `4626`; confirm with the user if the mapping is ambiguous) → build the issue URL from `redmine.url` (e.g. `<redmine.url>/issues/<id>`).
- `redmine.url` unset and no URL to go on → ask the user for the base URL or the direct issue link rather than guessing.

## `fetch(id)`

1. Load browser tools per [policies/browser](../policies/browser.md)
2. `tabs_context_mcp` first — if the issue is already open in an existing tab, reuse it; otherwise `navigate`/`tabs_create_mcp` to the resolved URL
3. `get_page_text` (or `read_page` if the DOM structure matters) to pull subject, description, status, and comments straight off the rendered page — Redmine renders the full comment/update history on one page by default, so a single call normally captures everything below the description too
4. If the page shows a login form instead of the issue, the session isn't authenticated in that tab — tell the user rather than guessing at content
5. (Rare) if the issue is unusually long and comments look truncated/paginated rather than fully rendered, that's a separate case not handled by the steps above — flag it rather than assuming you got the full history

This reuses the exact same browser/session as [phases/verify](../phases/verify.md) and this adapter's own writes below — one authenticated tab covers the whole pipeline.

**Field mapping** (contract fields on the left):
- `subject` ← the issue's subject line
- `description` ← the issue's description
- `status` ← Redmine's own status name; treat `Resolved`/`Closed`/`Rejected` as terminal, anything else as open
- `assignee` ← `assigned_to`; compare against `redmine.user` for the assignment check
- `comments[]` ← the journal/update history, in order
- `attachments[]` ← thumbnails/links on the rendered issue page
- Extra: `tracker` / `priority` / `custom_fields` are also available if present — context, not required by the contract

## Attachments

Bug tickets and UI-change requests frequently carry the actual requirement in an attached image (screenshot of the bug, a design mockup) or a log file — the text description alone can be incomplete or even misleading without it. Don't skip attachments just because the description reads as sufficient on its own.

- `get_page_text` won't capture embedded images. If the issue page shows attached images/thumbnails, either take a screenshot of the rendered issue page or navigate to the attachment's direct URL (it's an authenticated page, same session) and capture it, then view it before implementing — especially for anything tagged bug/UI.
- A text/log attachment can usually be read straight off its rendered page the same way; if it's served as a raw download instead, view it in the browser tab rather than assuming its content from the filename.
- If an attachment fails to load or its content doesn't match what the description says, treat that as exactly the kind of ambiguity [policies/safety](../policies/safety.md) says is worth stopping and asking about, not guessing past.

## `write_comment(id, text)` / `set_status(id, status_key)`

1. Navigate to (or reuse) the issue's update form via `claude-in-chrome`
2. Fill the comment/notes field with the message
3. If `status_key` is `in_progress` or `in_review`, set the status from the dropdown on the same form — `in_progress_status_id`/`review_status_id` in the config are IDs, but the dropdown itself shows the actual status *names*; match by name rather than needing the numeric ID. If `in_progress_mode` is `comment` (or `skip`), don't touch the status dropdown for `in_progress` — a plain comment (or nothing) is the intended behavior. If neither the config nor the dropdown makes the right status obvious, ask the user rather than guessing which one to pick.
4. Submit, then re-read the page to confirm the comment/status actually landed before reporting success
