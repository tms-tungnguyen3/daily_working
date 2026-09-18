# Policy: Browser Usage (claude-in-chrome)

Used unconditionally by [phases/verify](../phases/verify.md) — driving the real app under test is always a browser operation, regardless of which task tracker is configured. Also used by [adapters/redmine](../adapters/redmine.md) for every one of its operations (`fetch`, `write_comment`, `set_status`); **not** every adapter uses the browser at all — [adapters/github](../adapters/github.md) does the equivalent through the `gh` CLI instead. Check the active adapter file before assuming this policy applies to Phase 1/close-out writes on a given project.

## Loading tools

`claude-in-chrome` tools are deferred — load the ones a step needs in a single `ToolSearch` call rather than one at a time:

- Fetching a ticket page or writing to it (Redmine only — [adapters/redmine](../adapters/redmine.md)): `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__tabs_create_mcp`
- Verifying a change ([phases/verify](../phases/verify.md), every adapter): `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__tabs_close_mcp` (add `read_console_messages` / `read_network_requests` for debugging)

Always call `tabs_context_mcp` first — if a relevant tab is already open (the ticket, or the app under test) **and it belongs to this same task's own session**, reuse it instead of opening a new one. Running multiple tasks in parallel (see [policies/parallel](./parallel.md))? Don't reuse a tab that belongs to a different task's in-progress session — open a new one instead of guessing whose tab it is.

## One session covers Redmine end to end

On a project configured for [adapters/redmine](../adapters/redmine.md), fetching, writing, and verifying all use the exact same authenticated browser/session — the tracker login carries over to the app-under-test tab and back, and there's no separate credential setup anywhere in that adapter. This doesn't apply to adapters that don't use the browser for tracker I/O (e.g. [adapters/github](../adapters/github.md), which only needs `claude-in-chrome` for [phases/verify](../phases/verify.md)).

## Don't trigger dialogs

Avoid actions that raise a native `alert`/`confirm`/`prompt` — these block all further browser events and stop the extension from receiving any subsequent command. Skip buttons/links that trigger confirmation dialogs (e.g. destructive "Delete" actions) where a non-dialog path exists; if a dialog does fire, tell the user they need to dismiss it manually rather than continuing to retry commands.

## Recognize "not authenticated," don't guess past it

If a page shows a login form instead of the expected content, the session isn't authenticated in that tab — say so rather than inferring content from a partial page.

## When to stop and ask

If browser tool calls fail or return errors repeatedly (2-3 attempts), elements don't respond, or a page won't load, stop and ask the user how to proceed instead of continuing to retry the same action.
