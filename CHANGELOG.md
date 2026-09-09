# Changelog

All notable changes to the `daily-working` skill/plugin are documented here. Format loosely follows [Keep a Changelog](https://keepachangelog.com/); versions match `skills/daily-working/SKILL.md`'s frontmatter and `.claude-plugin/plugin.json`.

## [2.0.0] — 2026-09-09

### Changed — breaking (structure)

- **Split the single 335-line `SKILL.md` into a modular layout.** `SKILL.md` is now a short router (purpose, when-to-use, workflow picker, structure map); everything else moved into standalone files loaded on demand:
  - `workflows/` — `implement.md`, `review.md`, `resume.md` (entry points)
  - `phases/` — `setup.md`, `fetch-task.md`, `assess.md`, `implement.md`, `test.md`, `verify.md`, `close.md` (what each pipeline step does — now tracker-agnostic, see "Multi-tracker support" below)
  - `policies/` — `safety.md`, `git.md`, `database.md`, `browser.md`, `tracker-adapter.md` (cross-cutting rules previously repeated inline at each phase that needed them)
  - `adapters/` — `redmine.md`, `github.md` (one file per supported tracker; new in this release, see below)
  - `templates/` — `pr.md`, `tracker-write.md`, `summary.md` (literal text/structure to fill in, previously duplicated per phase)
  - **If installed as a personal or project skill, re-copy the whole `skills/daily-working/` directory, not just `SKILL.md`** — the old single-file copy step is no longer sufficient, since `SKILL.md` now links out to the files above. See the updated install instructions in `README.md`.
- **`pr.include_redmine_link` → `pr.include_tracker_link`** in `.claude/daily-working.yml`. Forced by the PR template no longer being able to assume Redmine's `## Redmine` section header. Existing configs setting the old key need to rename it; the semantics are unchanged (default `true`) so a Redmine project that never set it explicitly needs no change at all.

### Added — multi-tracker support

- **`policies/tracker-adapter.md`** — the interface every task tracker adapter implements: `fetch(id)`, `write_comment(id, text)`, `set_status(id, status_key)` (status_key is always one of exactly two abstract keys, `in_progress`/`in_review`), plus the normalized `fetch()` result shape (`subject`/`description`/`status`/`assignee`/`comments[]`/`attachments[]`). `phases/`, `workflows/`, and `templates/` call only this contract — none of them mention a specific tracker by name any more.
- **`adapters/redmine.md`** — the concrete Redmine mechanics (previously inline in `phases/fetch-task.md` and the old `templates/redmine-comment.md`), now implementing the contract above via the `claude-in-chrome` browser session.
- **`adapters/github.md`** — first new tracker since launch: GitHub Issues via the `gh` CLI (the same auth `templates/pr.md`'s `gh pr create` already needs, so no new credential setup). Maps `in_progress`/`in_review` to labels (`github.in_progress_label`/`github.in_review_label`) since GitHub Issues has no native status field; attachments are recovered by scanning `body`/comment markdown for embedded links, since GitHub Issues has no separate attachments array either.
- **`task_tracker` field in `.claude/daily-working.yml`** (`redmine` or `github`) selects which adapter's config block (`redmine:` / `github:`) and mechanics are active. Adding a third tracker is one new `adapters/<name>.md` file — no phase, workflow, or template changes.
- `templates/redmine-comment.md` → **`templates/tracker-write.md`**: same four message templates (ambiguity question, impact concern, in-progress marker, close-out comment), now phrased against the abstract `write_comment`/`set_status` calls instead of Redmine specifically.
- **`workflows/review.md`** — a lighter path for when code already exists and just needs checking against a ticket, instead of re-running the full implement pipeline.
- **`workflows/resume.md`** — an explicit path for continuing a pipeline that previously paused on an ambiguity ([`phases/fetch-task.md`](skills/daily-working/phases/fetch-task.md)) or impact ([`phases/assess.md`](skills/daily-working/phases/assess.md)) question once that question is answered on the ticket.
- **`CHANGELOG.md`** (this file).

### Removed

- **The Redmine API key path (`REDMINE_API_KEY`, the old Phase 1/close-out "Option B").** Every install of this skill uses `claude-in-chrome`, so the API-key fallback was dead weight and a second code path to keep in sync. [`adapters/redmine.md`](skills/daily-working/adapters/redmine.md) now fetches and writes back exclusively through the authenticated browser session. `redmine.url` (or `REDMINE_URL`) is kept, but only to build the issue link when the user gives a bare ticket ID — no credential is read or stored anywhere in the pipeline.

### Notes

Aside from the API-key removal and the tracker-adapter split, no behavioral change to the default end-to-end flow on an existing Redmine project (`workflows/implement.md`) — the escalation mechanism, database-safety rule, branch/commit conventions, and templates are also consolidated into single canonical files (`policies/`, `templates/`) instead of being restated at each phase that used them, which is the main practical difference if you'd memorized where a given rule used to live in the old `SKILL.md`.

## [1.7.0] — 2026-09-08

- **Added Phase 1.5 (sanity-check & impact assessment)**, run after the requirement is settled and before any branching/coding: check the requirement still makes sense against the actual codebase, judge blast radius (auth/payments/migrations/shared config, reach into shared code, effect on other users, disproportionate scope), and escalate high-impact or non-sensical tasks the same way Phase 1 escalates ambiguity.

## [1.6.0] — 2026-09-01

- **Phase 0 now reads the target repo's own `CLAUDE.md`/`CONTRIBUTING.md`/`README.md` first**, pre-filling the setup questionnaire from what the project already documents about itself, and only asking the user for fields still unclear.
- Standardized `README.md`: badges, Mermaid architecture diagram, quick start, use cases.

## [1.5.0] — 2026-08-25

- **Added branch strategy**: dedicated branch per ticket via `git.branch_format`, never committing to the default branch.
- **Added assignee/status check** before starting work (already-assigned-to-someone-else / already-closed tickets stop and ask instead of proceeding silently).
- **Added `review_status_id`** and the PR body template — the pipeline now moves a ticket to "in review" (not Resolved/Closed) once a PR is open and browser-verified.
- Fixed a marketplace validation warning by adding a marketplace-level plugin description.

## [1.4.0] — 2026-08-24

- **Ambiguity now escalates to a Redmine comment** (not just asked in-chat) and pauses the pipeline when the user doesn't know the answer either — the reporter/PM watches the ticket, not the chat.
- **Added the "in progress" marker**: the ticket transitions (or gets a comment) as soon as the requirement is settled, instead of Redmine only hearing from the pipeline once at the very end.

## [1.3.0] — 2026-08-23

- **Added Phase 4 (close the loop)**: post a close-out comment and move the ticket to "in review" once the browser check is confirmed.
- **Added attachment handling** in Phase 1 — screenshots/mockups/logs are downloaded (or viewed via browser) and read before implementing, since the text description alone can be incomplete or misleading.

## [1.2.0] — 2026-08-22

- Version-bump housekeeping to pick up three unversioned commits: uninstall docs, a fix to read the *full* comment history (not just the latest entry) when resolving the requirement, and a clarification that the Phase 0 config lives at each git repo's root, not a shared parent folder.

## [1.1.0] — 2026-08-22

- **Added Phase 0**: per-repo `.claude/daily-working.yml` config (Redmine URL, ticket key prefix, branch/commit format, conventions, test command, DB safety, PR method, dev server URL) so the skill adapts to a project without editing the skill itself.

## [1.0.0] — 2026-08-21

- Initial release: Phase 1 (fetch from Redmine) → Phase 2 (implement via Claude CLI) → Phase 3 (verify via `claude-in-chrome`).
