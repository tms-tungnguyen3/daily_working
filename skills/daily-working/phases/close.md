# Phase 4: Wrap Up

Part of [workflows/implement](../workflows/implement.md) and [workflows/review](../workflows/review.md). Runs after [phases/verify](./verify.md) confirms the change works.

## Report the summary

Fill in [templates/summary](../templates/summary.md) and report it back to the user before doing anything outward-facing.

## Open the PR

Once the user confirms the browser check looked right, open the PR per [templates/pr](../templates/pr.md) using `pr.method` from the config file (a companion skill/command, or plain `gh pr create` if unset).

## Close the loop on the task tracker

Don't stop at reporting to the user in this conversation — the ticket itself should reflect that the work happened, otherwise the next person to look at the tracker has no idea. **Ask the user for confirmation before writing to the tracker** (see [policies/safety](../policies/safety.md) — a comment/status change is outward-facing, same as opening a PR) — do this after they've confirmed the browser check, not before.

Once confirmed and the PR is open, `write_comment` the [templates/tracker-write](../templates/tracker-write.md) close-out message (commit reference, what was verified in the browser, the PR link) and, only if the config or the user says so, `set_status(id, "in_review")` — **not** a closed/terminal state. The code hasn't been reviewed or merged yet at this point; marking it fully done here would be premature and someone could act on that before the PR is actually in. See the active adapter (e.g. [adapters/redmine](../adapters/redmine.md), [adapters/github](../adapters/github.md)) for what `in_review` concretely does on this tracker.

If the user doesn't want the ticket touched automatically (some teams manage this manually after their own review), skip this and just leave the summary in chat — note that in `.claude/daily-working.yml` under a new freeform note in `conventions.notes` so future runs don't ask again.

## Final close is separate

**Final close (Resolved/Closed) is a separate, later step, outside this run** — it belongs after the PR is actually merged, not after local verification. If the user reports back later that the PR merged, that's a fresh, small ask ("mark #4626 resolved, the PR merged") rather than something this pipeline does automatically here.
