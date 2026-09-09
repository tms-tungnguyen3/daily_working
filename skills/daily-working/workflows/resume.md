# Workflow: Resume

Use this when a previous [workflows/implement](./implement.md) (or [workflows/review](./review.md)) run paused on an escalation — an ambiguity question ([phases/fetch-task](../phases/fetch-task.md)) or an impact/sanity concern ([phases/assess](../phases/assess.md)) posted as a comment on the task tracker per [policies/safety](../policies/safety.md) — and the user now says to continue (e.g. "resume #4626", "the comment got answered, keep going").

Never trust memory of the old ticket state — always re-fetch.

## Sequence

1. **[phases/fetch-task](../phases/fetch-task.md)** — re-fetch the ticket fresh. Read the new journal entries specifically for an answer to the question that was posted.
   - Still unanswered → report that back to the user and stay paused. Don't proceed on a guess just because time has passed.
   - Answered → extract the resolved requirement (the new comment is now the effective requirement, per [phases/fetch-task](../phases/fetch-task.md)'s "most recent substantive clarification wins" rule) and continue.
2. **Pick up where it left off**, based on which phase raised the escalation:
   - Paused during [phases/fetch-task](../phases/fetch-task.md) (ambiguity) → continue into **[phases/assess](../phases/assess.md)**, then the rest of [workflows/implement](./implement.md).
   - Paused during [phases/assess](../phases/assess.md) (impact/sanity concern) → re-run [phases/assess](../phases/assess.md)'s decision step against the clarified answer (it may change the impact judgment, not just unblock it), then continue into **[phases/implement](../phases/implement.md)** onward.
3. Continue the remaining phases exactly as [workflows/implement](./implement.md) (or [workflows/review](./review.md)) describes — nothing about the rest of the pipeline changes just because it was paused partway through.

If the user asks to resume a ticket but no prior escalation is on record in this conversation, treat it as a fresh [workflows/implement](./implement.md) run instead — check the ticket's actual current state rather than assuming what was paused.
