# Adapter: GitHub Issues

Implements the [policies/tracker-adapter](../policies/tracker-adapter.md) contract via the `gh` CLI — the same auth [templates/pr](../templates/pr.md) already relies on for `gh pr create`, so there's nothing new to authenticate. Run `gh auth status` once if a call below fails with an auth error, and ask the user to `gh auth login` rather than guessing at credentials.

## Config block

```yaml
github:
  repo:                     # "owner/repo", or omit to infer from `git remote get-url origin` in the target repo
  user:                      # GitHub username this pipeline runs as — used for the assignee check
  in_progress_label:        # in_progress mapping: label to add when work starts (see "No native status" below)
  in_review_label:          # in_review mapping: label to add once implementation is browser-verified and a PR is open
```

GitHub Issues have no status field beyond open/closed, so this adapter maps the two abstract status keys to **labels** rather than a state transition — see "No native status concept" below. `in_progress_label`/`in_review_label` must already exist on the repo (`gh label list --repo <owner>/<repo>`); if either is missing, create it once (`gh label create "<name>" --repo <owner>/<repo>`) or ask the user which existing label to reuse rather than guessing a name.

## Resolve what to open

- User pasted a full issue URL → `gh issue view <url>` accepts it directly, no repo flag needed.
- User gave a bare number (`123`) → needs `--repo owner/repo`: use `github.repo` from the config if set, otherwise infer it from `git remote get-url origin` in the target repo (parse `owner/repo` out of the remote URL).
- Neither a URL nor a resolvable repo → ask the user for the repo (`owner/repo`) rather than guessing.

## `fetch(id)`

```bash
gh issue view <number-or-url> [--repo <owner>/<repo>] --json title,body,state,assignees,labels,comments,url
```

**Field mapping** (contract fields on the left):
- `subject` ← `title`
- `description` ← `body`
- `status` ← `state` (`OPEN`/`CLOSED`) plus `labels` — treat `CLOSED` as terminal; `OPEN` with no workflow label as not-yet-started; `OPEN` with `in_progress_label`/`in_review_label` present as whatever that label says
- `assignee` ← `assignees[].login` (GitHub allows multiple; compare `github.user` against the list, not a single value, for the assignment check)
- `comments[]` ← `comments[]` (`author.login`, `body`, `createdAt`), in order
- `attachments[]` ← **GitHub Issues has no separate attachment list.** Images/files show up only as markdown links/embeds inside `body` and each comment's `body` (e.g. `![...](https://github.com/user-attachments/...)`). Scan those fields for such links and treat each as an attachment — see "Attachments" below.

## Attachments

Since there's no dedicated attachments array here, treat any image/file link embedded in `body` or a comment as an attachment:

- Fetch the linked URL directly (it's a public or session-authenticated asset URL depending on repo visibility) and view it before implementing — especially for anything that looks like a bug screenshot or design mockup.
- If a link 404s or its content doesn't match what the description says, that's exactly the kind of ambiguity [policies/safety](../policies/safety.md) says is worth stopping and asking about, not guessing past.

## `write_comment(id, text)`

```bash
gh issue comment <number-or-url> [--repo <owner>/<repo>] --body "<text>"
```

## `set_status(id, status_key)` — no native status concept

GitHub Issues doesn't have Redmine-style status IDs, so this maps `status_key` to a label add/remove instead of a real transition:

```bash
gh issue edit <number-or-url> [--repo <owner>/<repo>] --add-label "<in_progress_label or in_review_label>"
```

- Going from `in_progress` → `in_review`: also remove the now-stale label so the issue doesn't carry both at once:
  ```bash
  gh issue edit <number-or-url> [--repo <owner>/<repo>] --remove-label "<in_progress_label>"
  ```
- If `in_progress_label`/`in_review_label` is unset in the config, fall back to `write_comment` for that transition instead ("Starting work on this." / the close-out comment) rather than inventing a label name.
- Never use `gh issue close` for either abstract key — that's a real terminal close, and per the contract `in_review` is explicitly not a closed state. Final close (if this project even manages that from Redmine-style workflow at all) is the same later, separate, human-triggered step described in [phases/close](../phases/close.md).
