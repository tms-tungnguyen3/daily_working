# Phase 0: Project Setup (first run only)

Part of [workflows/implement](../workflows/implement.md). This skill is deliberately generic out of the box — Phase 0 is what makes it fit *this* project without editing the skill itself.

Before [phases/fetch-task](./fetch-task.md), check for a config file at `.claude/daily-working.yml` at the root of the **git repository actually being changed** (`.claude/daily-working.json` also accepted) — not at the Claude Code session's top-level working directory if that's a different, larger folder (e.g. a monorepo-style parent that isn't itself a git repo and just contains several independent repos). Each repo gets its own file: conventions, test framework/command, and coverage bar are frequently *not* shareable across repos even when they're worked on from the same parent folder — don't assume one config covers all of them.

- **Exists** → read it and use its values through every later phase instead of asking/guessing per field.
- **Missing** → this is the first time the skill runs in this repo.

  **First, check what the repo already documents about itself** — before asking the user anything, look for `CLAUDE.md`, `CONTRIBUTING.md`, `.claude/CLAUDE.md`, and the root `README.md` in the target repo, and skim any that exist for: architecture/authorization/i18n conventions, test framework and how to run it, coverage expectations, branch naming, commit message format, and DB safety notes. These are the project's own stated rules — prefer them over asking, and never contradict them. Pre-fill the questionnaire below with whatever they answer, and only ask the user for fields still unclear or unstated. When presenting the pre-filled answers back to the user for confirmation, say which file each one came from, so they can correct anything mis-read.

  Then ask the user a short setup questionnaire for whatever's left, and write the combined answers to `.claude/daily-working.yml` at that repo's root (create `.claude/` if needed):
  - **Which task tracker this project uses (`task_tracker`)** — pick from the adapters that exist under `adapters/` (today: `redmine` or `github`; see [policies/tracker-adapter](../policies/tracker-adapter.md)). This decides which adapter file's own questionnaire runs next — jump to it rather than asking generic tracker questions here:
    - `redmine` → [adapters/redmine](../adapters/redmine.md)'s config block (base URL, ticket key prefix, your username, in-progress/in-review status IDs)
    - `github` → [adapters/github](../adapters/github.md)'s config block (repo, your username, in-progress/in-review labels)
    - Neither fits → say so; adding a tracker means writing a new `adapters/<name>.md` per [policies/tracker-adapter](../policies/tracker-adapter.md), not forcing the answers into one of the two above
  - Branch naming convention (`git.branch_format`), or accept the default `{ticket_key}-{slug}` — see [policies/git](../policies/git.md)
  - Commit message format/template and max subject length — see [policies/git](../policies/git.md)
  - Architecture, authorization, and i18n conventions to follow (a few keywords is enough, e.g. "Rails HMVC, Pundit, Rails I18n")
  - Test framework, the exact command to run it (e.g. `RAILS_ENV=test bundle exec rspec`), and the coverage bar expected — see [phases/test](./test.md)
  - Whether the `development` DB is a shared instance others depend on, plus any project-specific safety note to carry forward — see [policies/database](../policies/database.md)
  - How PRs get opened in this project (a companion skill/command name, or plain `gh pr create`) — see [templates/pr](../templates/pr.md)
  - Default local dev server URL, if there's a fixed one

  Leave anything the user doesn't know blank/null — a missing field just means "fall back to asking/inferring in the moment" for that one field, not a blocker for setup.

## Config file shape

YAML; JSON with the same keys works too:

```yaml
task_tracker: redmine       # "redmine" or "github" today — picks which adapters/*.md file's config block below is actually read; see policies/tracker-adapter.md

redmine:                    # present only when task_tracker: redmine — see adapters/redmine.md for every field
  url:
  ticket_key_prefix:
  user:
  in_progress_status_id:
  in_progress_mode:
  review_status_id:

# github:                   # present only when task_tracker: github — see adapters/github.md for every field
#   repo:
#   user:
#   in_progress_label:
#   in_review_label:

git:
  branch_format: "{ticket_key}-{slug}"   # branch name pattern; {ticket_key} = numeric id or prefixed key, {slug} = short kebab-case slug from the subject

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
  include_tracker_link: true   # whether the PR body should link back to the ticket

dev_server:
  default_url:
```

This file holds conventions, not secrets — safe to commit so the whole team gets the same setup answers. Neither shipped adapter needs a credential stored here: [adapters/redmine](../adapters/redmine.md) authenticates via the already-logged-in `claude-in-chrome` browser session, [adapters/github](../adapters/github.md) via the `gh` CLI's own auth. A future adapter that genuinely needs an API key should keep it as an env var, the same way both of these avoid storing one.

Only the block matching `task_tracker` gets read — [phases/fetch-task](./fetch-task.md) and [templates/tracker-write](../templates/tracker-write.md) dispatch to `adapters/<task_tracker>.md`, which is the only place that knows its own field names and what they mean.

If the user later says a convention changed, update the relevant field in this file rather than only fixing it in the moment — that's what keeps Phase 0 from being asked again unnecessarily.
