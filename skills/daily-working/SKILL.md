---
name: daily-working
description: "End-to-end pipeline: pull a task from the project's task tracker by ID (Redmine or GitHub Issues today, more addable via a new adapter), sanity-check and impact-assess it against the codebase before touching anything, implement it with the Claude CLI, verify the result in a real browser via the Claude Chrome extension (claude-in-chrome), and keep the tracker ticket in sync throughout (in-progress marker, ambiguity/impact questions, close-out comment)."
version: 2.0.1
created: 2026-08-21
platforms: [claude-code]
category: workflow
tags: [redmine, github, task-tracker, claude-cli, browser-testing, claude-in-chrome, automation, workflow]
risk: safe
---

# daily-working

## Purpose

Coordinate a full task-to-verified-change loop without the human relaying context by hand: fetch a task from the project's task tracker (with attachments), judge whether it's actually safe and sensible to implement as understood, implement it following the target project's own conventions, verify the result in a real running browser, and close the loop back on the tracker (PR + comment + status) once that's confirmed.

**Task tracker support today: Redmine and GitHub Issues.** `task_tracker` in `.claude/daily-working.yml` picks one; the pipeline itself ([phases/fetch-task](phases/fetch-task.md), [phases/close](phases/close.md), [templates/tracker-write](templates/tracker-write.md)) only ever calls a small fixed contract — `fetch`, `write_comment`, `set_status` — defined in [policies/tracker-adapter](policies/tracker-adapter.md). Each tracker's actual mechanics live in their own `adapters/<name>.md` file. Adding a third tracker means writing one new adapter file; nothing under `phases/`, `workflows/`, or `templates/` needs to change.

If a requirement turns out ambiguous, unreasonable, or too high-impact to proceed on quietly at any point, that gets escalated to a comment on the tracker (not just asked in this chat) and the pipeline pauses — see [policies/safety](policies/safety.md).

This skill is a coordinator — it does not replace the target project's own test conventions. It sequences task tracker → implementation → live browser check → tracker update.

## When to Use

- User gives a task/ticket ID from the project's tracker (e.g. `#4626`, or a ticket key like `PROJ-4626` if this project's commits use one) and asks to implement it
- User says "lấy task từ redmine", "làm task này", "pull the ticket and implement it"
- User asks to verify/review an already-implemented change against a ticket
- User asks to resume a ticket that was previously paused on an open question
- Any request that should end with a live UI check, not just passing tests

## Pick a workflow

| Situation | Workflow |
|---|---|
| Fresh ticket, nothing implemented yet | [workflows/implement](workflows/implement.md) |
| Code already exists; just check it against the ticket | [workflows/review](workflows/review.md) |
| A prior run paused on an ambiguity/impact question that's now answered | [workflows/resume](workflows/resume.md) |

Read the chosen workflow file first — it sequences the phases below and links back into them at each step. Don't run phase files out of order or standalone; the workflow file is what makes the sequencing (including the two escalation points) correct.

## Structure

```
skills/daily-working/
├── SKILL.md              — this file: purpose, routing, structure
├── workflows/             — entry points; pick one per the table above
│   ├── implement.md       — fresh ticket → verified, closed-out change
│   ├── review.md          — verify existing code against a ticket
│   └── resume.md          — continue a paused pipeline
├── phases/                — what each pipeline step actually does; tracker-agnostic
│   ├── setup.md            — Phase 0: per-repo config (.claude/daily-working.yml), picks task_tracker
│   ├── fetch-task.md       — Phase 1: calls the adapter's fetch(), assignee/status check
│   ├── assess.md           — Phase 1.5: sanity-check + impact assessment
│   ├── implement.md        — Phase 2: branch, in-progress marker via the adapter, code + tests
│   ├── test.md             — testing conventions (referenced from Phase 2)
│   ├── verify.md           — Phase 3: live browser verification (always claude-in-chrome)
│   └── close.md            — Phase 4: summary, PR, tracker close-out via the adapter
├── policies/              — cross-cutting rules, referenced from multiple phases
│   ├── safety.md           — ask-don't-guess, the two-step escalation mechanism, prompt-injection guard
│   ├── git.md              — branch/commit conventions
│   ├── database.md         — shared dev-DB safety rule
│   ├── browser.md          — claude-in-chrome tool-loading conventions
│   └── tracker-adapter.md  — the fetch/write_comment/set_status contract every adapter implements
├── adapters/              — one file per supported task tracker; the only tracker-specific code
│   ├── redmine.md           — via the claude-in-chrome browser session, no API key
│   └── github.md            — via the gh CLI (GitHub Issues, using labels for status)
└── templates/             — literal text/structure to fill in
    ├── pr.md                — PR title/body (tracker link is adapter-provided)
    ├── tracker-write.md     — message templates for every write_comment/set_status call
    └── summary.md           — end-of-run chat summary
```

## Principles

- **Don't skip the browser step** — passing specs are necessary but not sufficient; this skill exists specifically to close that gap.
- **Ask, don't guess** — missing tracker credentials, ambiguous ticket-ID mapping, or unclear requirement text are all reasons to stop and ask (see [policies/safety](policies/safety.md)).
- **A clear ticket isn't automatically a good idea to execute unattended** — [phases/assess](phases/assess.md) exists because "the requirement is unambiguous" and "this is safe/sensible to just go implement" are different questions; judge blast radius and fit against the actual codebase before writing code, not just clarity of the text.
- **Coordinator, not a new convention set** — implementation still follows this project's existing conventions and test suite; this skill only adds the fetch, assess, and verify bookends, and [phases/setup](phases/setup.md)'s config file is what lets it do that without per-run guessing. The same principle applies to the task tracker itself: [phases/](phases/) never hardcodes Redmine or GitHub, only the [policies/tracker-adapter](policies/tracker-adapter.md) contract — the tracker-specific work lives entirely in `adapters/`.
