# Phase 1.5: Assess Impact & Sanity-Check

Part of [workflows/implement](../workflows/implement.md). Runs after [phases/fetch-task](./fetch-task.md)'s requirement is settled (no open ambiguity) and before [phases/implement](./implement.md)'s branch/in-progress step — never after.

[phases/fetch-task](./fetch-task.md) settles *what* the requirement says. This phase judges *whether it should be implemented as understood, and how carefully* — before branching, before writing a line of code. Skipping straight from "requirement is clear" to "let's build it" is exactly the gap this phase closes: a clear requirement can still be a bad or oversized idea to execute unattended.

## 1. Sanity-check against the actual codebase

A requirement can be perfectly unambiguous and still not make sense to implement as literally stated:

- The behavior described already exists, or was already fixed by a more recent, unrelated change — implementing it again would be redundant or would reintroduce something.
- The described approach contradicts this project's current architecture/conventions in a way that can't be reconciled without a design decision bigger than the ticket implies (e.g. the ticket assumes a data model or flow the code doesn't actually have).
- The acceptance criteria, read literally, can't actually be satisfied by a change scoped to what the ticket describes (it would require touching systems the ticket doesn't mention at all).

This is different from [phases/fetch-task](./fetch-task.md) ambiguity: the text isn't unclear, it's just no longer (or never was) a good match for what the codebase actually does. Use judgment based on what's actually in the repo — read the relevant code before concluding this, don't assume from the ticket text alone.

## 2. Assess impact / blast radius

Before deciding this is safe to just go implement, weigh the risk signals in [policies/safety](../policies/safety.md) (auth/payments/migrations/shared config, reach into shared code, effect on users beyond the reporter, disproportionate scope).

## 3. Decide

- **Low impact, sanity-checks out** → proceed straight to [phases/implement](./implement.md), no need to ask — most routine fixes/small features land here.
- **High impact (any risk signal) or fails the sanity check** → stop, don't branch or write code yet. Resolve it via the same two-step mechanism as [phases/fetch-task](./fetch-task.md) ambiguity (see [policies/safety](../policies/safety.md)): ask in-chat first, stating plainly what's risky or what doesn't add up; if the user isn't sure either, post the concern per [templates/tracker-write](../templates/tracker-write.md) and stop the pipeline. Don't guess and proceed on a high-impact or seemingly-unreasonable task just because the ticket text was clear.
- Note the outcome in the [phases/close](./close.md) summary either way — even a low-impact task gets a one-line note on why it was judged safe to proceed, so the human reviewing later sees the assessment happened, not just its silence.
