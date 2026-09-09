# Workflow: Implement

The default, full pipeline. Use this when the user hands over a fresh task/ticket ID (or issue URL) from the project's task tracker and asks to implement it — this is what "pull task from Redmine and implement it" means today (Redmine being the only tracker this skill can currently fetch from/write back to).

## Sequence

1. **[phases/setup](../phases/setup.md)** — first run in this repo only; every later run reads the saved `.claude/daily-working.yml` instead.
2. **[phases/fetch-task](../phases/fetch-task.md)** — pull the ticket + attachments, check assignee/status.
   - Requirement ambiguous or contradicts the codebase? → escalate per [policies/safety](../policies/safety.md) and **stop**. Resume later via [workflows/resume](./resume.md).
3. **[phases/assess](../phases/assess.md)** — sanity-check and impact-assess before touching code.
   - High impact or fails the sanity check? → escalate per [policies/safety](../policies/safety.md) and **stop**. Resume later via [workflows/resume](./resume.md).
4. **[phases/implement](../phases/implement.md)** — branch, mark "in progress", write the change and tests (mind [policies/database](../policies/database.md)), commit.
5. **[phases/verify](../phases/verify.md)** — drive the real app in a browser; don't report success on assumed behavior.
6. **[phases/close](../phases/close.md)** — summary, PR, Redmine close-out (with confirmation).

Each phase file is the source of truth for its step — read it when you reach it rather than relying on this summary alone.

## Principles

- **Don't skip the browser step** — passing specs are necessary but not sufficient; this skill exists specifically to close that gap.
- **Ask, don't guess** — see [policies/safety](../policies/safety.md).
- **A clear ticket isn't automatically a good idea to execute unattended** — [phases/assess](../phases/assess.md) exists because "the requirement is unambiguous" and "this is safe/sensible to just go implement" are different questions.
- **Coordinator, not a new convention set** — implementation still follows this project's existing conventions and test suite; this skill only adds the fetch, assess, and verify bookends, and [phases/setup](../phases/setup.md)'s config file is what lets it do that without per-run guessing.
