---
name: daily-working
description: "End-to-end pipeline: pull a task from Redmine by ID, implement it with the Claude CLI, then verify the result in a real browser via the Claude Chrome extension (claude-in-chrome)."
version: 1.0.0
created: 2026-08-21
platforms: [claude-code]
category: workflow
tags: [redmine, claude-cli, browser-testing, claude-in-chrome, automation, workflow]
risk: safe
---

# daily-working

## Purpose

Coordinate a full task-to-verified-change loop without the human relaying context by hand:

1. **Fetch** the task/ticket from Redmine by ID.
2. **Implement** the change with the Claude CLI (this agent, following the target project's existing conventions and test suite directly).
3. **Verify** the change in a real running UI using the `claude-in-chrome` extension, driving the actual browser instead of trusting code review alone.

This skill is a coordinator — it does not replace the target project's own test conventions. It sequences Redmine → implementation → live browser check.

## When to Use

- User gives a Redmine task/ticket ID (e.g. `#4626`, or a ticket key like `PROJ-4626` if this project's commits use one) and asks to implement it
- User says "lấy task từ redmine", "làm task này", "pull the ticket and implement it"
- Any request that should end with a live UI check, not just passing tests

---

## Phase 1: Fetch the Task from Redmine

Two ways to get the task in, pick whichever fits the situation:

### Option A — Browser session already logged in (preferred, no credentials needed)

If the user pastes an issue URL and has a Chrome session already authenticated against Redmine, just read the page directly — no API key required:

1. Load browser tools: `ToolSearch` with `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__tabs_create_mcp`
2. `tabs_context_mcp` first — if the issue is already open in an existing tab, reuse it; otherwise `navigate`/`tabs_create_mcp` to the pasted URL
3. `get_page_text` (or `read_page` if the DOM structure matters) to pull subject, description, status, and comments straight off the rendered page
4. If the page shows a login form instead of the issue, the session isn't authenticated in that tab — tell the user rather than guessing at content

This reuses the exact same browser/session as Phase 3, so there's no separate credential setup.

### Option B — API key (fallback, no browser session available)

**Config required** (ask the user if missing — do not guess):
- `REDMINE_URL` — base URL of the Redmine instance
- `REDMINE_API_KEY` — personal API key (Redmine → "My account" → API access key)

**Resolve the ID:**
- Numeric ID (`4623`) → use directly
- Ticket key seen in commits (e.g. `PROJ-4626`) → strip the prefix to get the numeric Redmine issue ID (`4626`); confirm with the user if the mapping is ambiguous
- Or lift the numeric ID straight out of a pasted issue URL (`.../issues/4626`)

**Fetch:**

```bash
curl -s -H "X-Redmine-API-Key: $REDMINE_API_KEY" \
  "$REDMINE_URL/issues/<id>.json?include=journals,attachments"
```

(Swap `curl` for whatever HTTP client/CLI wrapper this environment provides.)

### Extract (either option)

- `subject` — becomes the basis of the commit message subject (format it per this project's commit convention, e.g. `[<ticket-key>]: <subject>` or plain `<subject>`)
- `description` — the requirement text to implement
- `tracker` / `status` / `priority` — context, not blocking
- `custom_fields` — check for anything implementation-relevant (target module, environment)
- `journals` / comments — recent ones may contain clarifications or scope changes; skim the latest

If the description is ambiguous or contradicts what the codebase currently does, stop and ask — don't guess.

---

## Phase 2: Implement via Claude CLI

Implement the change directly:

1. Follow this project's existing architecture, authorization, and i18n conventions — match the surrounding code rather than introducing a new pattern
2. Write/adjust tests to match this project's test conventions and coverage bar — put specs alongside the existing spec files for the module you touched, following that module's existing spec structure/factories rather than starting a new one
3. Commit using this project's commit message convention, built from the Redmine `subject` field, under 50 chars

Do not mark the task done yet — implementation is only verified by tests until Phase 3 confirms it in a real browser.

### ⚠️ Database safety — read before running any spec

`development` in `config/database.yml` points at a **shared** dev DB that multiple people/environments depend on staying intact. `test` should point at a local/isolated DB that's yours to use freely — reset, seed, wipe, whatever. (Exact host/credentials vary per project — check that project's `config/database.yml` rather than assuming.)

This restriction is specifically about **running the test suite** — it must never connect to the shared dev DB:
1. **Always run with the test environment explicit** — e.g. `RAILS_ENV=test bundle exec rspec ...` for Rails/rspec, or the equivalent env flag for whatever stack/test runner this project uses. Don't rely on a default that might resolve to `development`.
2. **Verify the resolved DB config isn't the shared/dev host** before the first run in a new shell/session — check `RAILS_ENV` and, if unsure, print the active DB config (`rails runner -e test 'puts ActiveRecord::Base.connection_db_config.configuration_hash'` or just eyeball `config/database.yml`'s `test:` block) and confirm the host matches the local `test` entry, not the `development` one.
3. **Never let a spec run touch the dev DB.** No migrations, seeds, console mutations, or ad-hoc scripts against `development`/the shared host via `rspec` — even for "just checking something." If a check requires touching data, do it against `test` or ask the user first.

This does **not** apply to Phase 3 — browser-driven E2E verification against the running dev server (and its dev DB) is expected and fine, no special caution needed there.

---

## Phase 3: Verify via Claude Chrome Extension

Use `claude-in-chrome` to drive the actual running app instead of relying solely on RSpec/system specs.

1. Confirm the dev server is running (ask the user for the local URL if unknown)
2. Load browser tools: `ToolSearch` with `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__tabs_close_mcp` (add `read_console_messages` / `read_network_requests` for debugging)
3. Call `tabs_context_mcp` first, then open/reuse a tab and navigate to the page affected by the Redmine task
4. Exercise the exact flow described in the ticket (click through, fill forms, trigger the changed behavior)
5. Check for: correct rendered output, no console errors, expected network responses (`render json:` payloads), and that the original bug/request from Redmine is actually resolved
6. If something's off, go back to Phase 2 — don't report success on stubbed/assumed behavior

---

## Phase 4: Wrap Up

Summary to report back:

```
Redmine task: <id> — <subject>
Implemented: {files changed}
Tests: {pass/fail, coverage}
Browser check: {what was exercised, what was observed}
Commit: {commit message used}
```

Next steps: open the PR once the user confirms the browser check looked right (use this project's PR-description convention or skill, if it has one).

---

## Principles

- **Don't skip the browser step** — passing specs are necessary but not sufficient; this skill exists specifically to close that gap
- **Ask, don't guess** — missing Redmine credentials, ambiguous ticket ID mapping, or unclear requirement text are all reasons to stop and ask
- **Coordinator, not a new convention set** — implementation still follows this project's existing conventions and test suite; this skill only adds the fetch and verify bookends

