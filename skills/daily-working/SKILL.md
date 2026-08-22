---
name: daily-working
description: "End-to-end pipeline: pull a task from Redmine by ID, implement it with the Claude CLI, then verify the result in a real browser via the Claude Chrome extension (claude-in-chrome)."
version: 1.2.0
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

## Phase 0: Project Setup (first run only)

This skill is deliberately generic out of the box — Phase 0 is what makes it fit *this* project without editing the skill itself.

Before Phase 1, check for a config file at `.claude/daily-working.yml` at the root of the **git repository actually being changed** (`.claude/daily-working.json` also accepted) — not at the Claude Code session's top-level working directory if that's a different, larger folder (e.g. a monorepo-style parent that isn't itself a git repo and just contains several independent repos). Each repo gets its own file: conventions, test framework/command, and coverage bar are frequently *not* shareable across repos even when they're worked on from the same parent folder — don't assume one config covers all of them.

- **Exists** → read it and use its values through Phases 1–4 instead of asking/guessing per field.
- **Missing** → this is the first time the skill runs in this repo. Ask the user a short setup questionnaire, then write the answers to `.claude/daily-working.yml` at that repo's root (create `.claude/` if needed):
  - Redmine base URL (or note that `REDMINE_URL`/`REDMINE_API_KEY` env vars will be used instead — never put credentials in this file)
  - Ticket key prefix used in commits, if any (e.g. `PROJ` for `PROJ-1234`), or "none — plain numeric IDs"
  - Commit message format/template and max subject length
  - Architecture, authorization, and i18n conventions to follow (a few keywords is enough, e.g. "Rails HMVC, Pundit, Rails I18n")
  - Test framework, the exact command to run it (e.g. `RAILS_ENV=test bundle exec rspec`), and the coverage bar expected
  - Whether the `development` DB is a shared instance others depend on, plus any project-specific safety note to carry forward
  - How PRs get opened in this project (a companion skill/command name, or plain `gh pr create`)
  - Default local dev server URL, if there's a fixed one

  Leave anything the user doesn't know blank/null — a missing field just means "fall back to asking/inferring in the moment" for that one field, not a blocker for setup.

**Config file shape** (YAML; JSON with the same keys works too):

```yaml
redmine:
  url:                     # or omit if using REDMINE_URL / REDMINE_API_KEY env vars
  ticket_key_prefix:       # e.g. "PROJ", or omit if tickets are plain numeric IDs

commit:
  format: "[{ticket_key}]: {subject}"
  max_subject_length: 50

conventions:
  notes:                   # freeform — architecture / authorization / i18n keywords

tests:
  framework:
  run_command:
  coverage_target:

database_safety:
  shared_dev_db: false      # true if `development` points at a DB others depend on
  notes:

pr:
  method:                   # e.g. "@pr-description", "gh pr create"

dev_server:
  default_url:
```

This file holds conventions, not secrets — safe to commit so the whole team gets the same setup answers. Never write API keys or credentials into it; those stay as env vars (Phase 1, Option B).

If the user later says a convention changed, update the relevant field in this file rather than only fixing it in the moment — that's what keeps Phase 0 from being asked again unnecessarily.

---

## Phase 1: Fetch the Task from Redmine

Two ways to get the task in, pick whichever fits the situation:

### Option A — Browser session already logged in (preferred, no credentials needed)

If the user pastes an issue URL and has a Chrome session already authenticated against Redmine, just read the page directly — no API key required:

1. Load browser tools: `ToolSearch` with `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__tabs_create_mcp`
2. `tabs_context_mcp` first — if the issue is already open in an existing tab, reuse it; otherwise `navigate`/`tabs_create_mcp` to the pasted URL
3. `get_page_text` (or `read_page` if the DOM structure matters) to pull subject, description, status, and comments straight off the rendered page — Redmine renders the full comment/update history on one page by default, so a single call normally captures everything below the description too
4. If the page shows a login form instead of the issue, the session isn't authenticated in that tab — tell the user rather than guessing at content
5. (Rare) if the issue is unusually long and comments look truncated/paginated rather than fully rendered, that's a separate case not handled by the steps above — flag it rather than assuming you got the full history

This reuses the exact same browser/session as Phase 3, so there's no separate credential setup.

### Option B — API key (fallback, no browser session available)

**Config required** — check `redmine.url` in `.claude/daily-working.yml` first; ask the user only for whatever's still missing (do not guess):
- `REDMINE_URL` — base URL of the Redmine instance (env var; falls back to `redmine.url` in the config file)
- `REDMINE_API_KEY` — personal API key (Redmine → "My account" → API access key) — always an env var, never stored in the config file

**Resolve the ID:**
- Numeric ID (`4623`) → use directly
- Ticket key seen in commits (uses `redmine.ticket_key_prefix` from the config file, e.g. `PROJ-4626`) → strip the prefix to get the numeric Redmine issue ID (`4626`); confirm with the user if the mapping is ambiguous
- Or lift the numeric ID straight out of a pasted issue URL (`.../issues/4626`)

**Fetch:**

```bash
curl -s -H "X-Redmine-API-Key: $REDMINE_API_KEY" \
  "$REDMINE_URL/issues/<id>.json?include=journals,attachments"
```

(Swap `curl` for whatever HTTP client/CLI wrapper this environment provides.)

### Extract (either option)

- `subject` — becomes the basis of the commit message subject (formatted per `commit.format` in the config file)
- `description` — the requirement text to implement
- `tracker` / `status` / `priority` — context, not blocking
- `custom_fields` — check for anything implementation-relevant (target module, environment)
- `journals` / comments — read the **full** history in order, not just the latest entry. Requirements often get refined or corrected after the initial description. When a later comment conflicts with the description or with an earlier comment, treat the most recent substantive clarification as the current, effective requirement — not the original description.

If the description is ambiguous, the comment thread has unresolved back-and-forth that doesn't clearly settle on a final requirement, or any of it contradicts what the codebase currently does, stop and ask — don't guess which version is authoritative.

---

## Phase 2: Implement via Claude CLI

Implement the change directly:

1. Follow the conventions recorded in `conventions.notes` (config file) — or this project's existing conventions if that field is empty — match the surrounding code rather than introducing a new pattern
2. Write/adjust tests using `tests.run_command` and `tests.coverage_target` from the config file — or infer them from the project if unset. Put specs alongside the existing spec files for the module you touched, following that module's existing spec structure/factories rather than starting a new one
3. Commit using `commit.format` from the config file, built from the Redmine `subject` field, staying within `commit.max_subject_length` chars (default 50)

Do not mark the task done yet — implementation is only verified by tests until Phase 3 confirms it in a real browser.

### ⚠️ Database safety — read before running any spec

Check `database_safety.shared_dev_db` in the config file first.

`development` in a project's DB config commonly points at a **shared** dev DB that multiple people/environments depend on staying intact, while `test` points at a local/isolated DB that's yours to use freely — reset, seed, wipe, whatever. If `database_safety.shared_dev_db` is `true`, treat that as confirmed for this project and also follow `database_safety.notes`. If it's `false`/unset, this restriction may not apply here — but still verify which DB the test environment resolves to before the first run in a new shell/session, since defaults vary per project.

This restriction is specifically about **running the test suite** — it must never connect to the shared dev DB:
1. **Always run with the test environment explicit** — e.g. `RAILS_ENV=test bundle exec rspec ...` for Rails/rspec, or the equivalent env flag for whatever stack/test runner this project uses (`tests.run_command` in the config file, if set). Don't rely on a default that might resolve to `development`.
2. **Verify the resolved DB config isn't the shared/dev host** before the first run in a new shell/session — print or eyeball the active test DB config and confirm it's the local/isolated one, not the `development` one.
3. **Never let a spec run touch the dev DB.** No migrations, seeds, console mutations, or ad-hoc scripts against `development`/the shared host via the test suite — even for "just checking something." If a check requires touching data, do it against the test DB or ask the user first.

This does **not** apply to Phase 3 — browser-driven E2E verification against the running dev server (and its dev DB) is expected and fine, no special caution needed there.

---

## Phase 3: Verify via Claude Chrome Extension

Use `claude-in-chrome` to drive the actual running app instead of relying solely on the test suite/system specs.

1. Confirm the dev server is running (use `dev_server.default_url` from the config file, or ask the user for the local URL if unset)
2. Load browser tools: `ToolSearch` with `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__tabs_close_mcp` (add `read_console_messages` / `read_network_requests` for debugging)
3. Call `tabs_context_mcp` first, then open/reuse a tab and navigate to the page affected by the Redmine task
4. Exercise the exact flow described in the ticket (click through, fill forms, trigger the changed behavior)
5. Check for: correct rendered output, no console errors, expected network responses, and that the original bug/request from Redmine is actually resolved
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

Next steps: open the PR once the user confirms the browser check looked right, using `pr.method` from the config file (a companion skill/command, or plain `gh pr create` if unset).

---

## Principles

- **Don't skip the browser step** — passing specs are necessary but not sufficient; this skill exists specifically to close that gap
- **Ask, don't guess** — missing Redmine credentials, ambiguous ticket ID mapping, or unclear requirement text are all reasons to stop and ask
- **Coordinator, not a new convention set** — implementation still follows this project's existing conventions and test suite; this skill only adds the fetch and verify bookends, and Phase 0's config file is what lets it do that without per-run guessing
