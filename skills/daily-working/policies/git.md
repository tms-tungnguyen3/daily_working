# Policy: Git Conventions

Used by [phases/implement](../phases/implement.md).

## Never commit to the default branch

Never commit directly to the repo's default branch (`main`/`master`/whatever `git symbolic-ref` reports) — always work on a dedicated branch, even for a small fix.

If already on a non-default branch when [phases/implement](../phases/implement.md) starts (e.g. the user already switched), it's fine to keep using it — just confirm it isn't the default branch before committing.

## Branch naming

Name the branch from `git.branch_format` in `.claude/daily-working.yml` (default `{ticket_key}-{slug}`, e.g. `4626-fix-login-redirect`), where `{slug}` is a short kebab-case slug derived from the ticket `subject`. Create and check it out before touching any files:

```bash
git checkout -b <branch-name>
```

## Commit message format

Build the commit message from `commit.format` in the config file (default `[{ticket_key}]: {subject}`), using the tracker's `subject` field, staying within `commit.max_subject_length` chars (default 50).
