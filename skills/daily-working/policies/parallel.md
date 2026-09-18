# Policy: Running Multiple Tasks in Parallel

This pipeline defaults to one task at a time in the repo's normal working directory. That's fine for sequential work, but it breaks the moment a second task starts before the first is committed: only one branch can be checked out at once, so `git checkout -b` for task B either fails or silently carries task A's uncommitted files into task B's branch. Two tasks in the same directory are never actually isolated — don't attempt it.

## Isolate by git worktree, not by directory discipline

Before [phases/implement](../phases/implement.md) creates a branch for a task, check whether any other task from this skill is already in flight (uncommitted changes on another branch, or the user says they want to start task B while task A isn't closed out yet). If so, give that task its own **git worktree** instead of branching in place.

Most setups have several independent repos checked out under one non-git parent folder (per [phases/setup](../phases/setup.md)'s "monorepo-style parent" note) — e.g. `workspace/repo1/`, `workspace/repo2/`. Don't scatter worktrees as loose siblings of those repos; that mixes real repo checkouts with transient task dirs at the same level. Instead, keep one dedicated worktree directory at the parent-folder level and namespace each worktree by repo + ticket, so it stays unambiguous even with several repos' tasks running at once:

```
workspace/
├── repo1/                        ← the repo's normal default-branch checkout
├── repo2/
└── .worktrees/
    ├── repo1-4626-fix-login/     ← task #4626 on repo1
    ├── repo1-4700-add-export/    ← a second, concurrent task on repo1
    └── repo2-1189-oauth-bug/     ← a task on repo2
```

Resolve the target directory from `parallel.worktree_dir` in that repo's `.claude/daily-working.yml` (a path relative to the repo root) if set, otherwise fall back to the default `../.worktrees` silently — no need to ask, unless the user later says they want it somewhere else, in which case save that to the config file the same way any other convention change is saved. Create it with:

```bash
git worktree add "$(git rev-parse --show-toplevel)/../.worktrees/<repo-name>-<ticket-key>" -b <branch-name>
```

(substitute the resolved `parallel.worktree_dir` for `../.worktrees` if the config sets something else)

- Run this from the repo's default branch checkout, not from inside another task's worktree.
- Everything downstream — [phases/implement](../phases/implement.md)'s branch/commit, [phases/test](../phases/test.md), [phases/verify](../phases/verify.md) — then runs with that worktree directory as the working directory for that task, exactly as it would in a normal checkout. Nothing in those phases changes; only *which directory* they operate in changes.
- `.claude/daily-working.yml` is a tracked file, so every worktree of the same repo sees the same config automatically — no per-worktree setup needed.
- Dependencies are not shared automatically: run this project's install step (`npm install`, `bundle install`, etc.) once inside the new worktree before [phases/test](../phases/test.md) or [phases/verify](../phases/verify.md) run there. Don't symlink `node_modules`/`vendor` from another worktree — two tasks may edit dependency manifests differently mid-flight.
- If [phases/verify](../phases/verify.md) needs a locally running dev server, each worktree needs its **own server instance on its own port** (or the servers run one at a time). Two worktrees pointed at the same dev server would both be exercising whichever branch happens to be running at that moment, not their own. Track the resolved per-worktree URL in the task's own notes for that run — this is a per-run detail, not something to add to the shared config file.
- Clean up with `git worktree remove <path>` once [phases/close](../phases/close.md) is done and the branch is pushed/merged — don't leave stale worktrees behind.

## One task per Claude Code session

A single session/context still only makes sense to advance one task at a time — running two tasks' phases interleaved in one conversation is how state leaks between them (wrong branch assumed current, wrong ticket's context used for a commit message). Drive each parallel task from its **own Claude Code session**, pointed at that task's worktree directory. This skill's phases don't need to know sessions are parallel; the isolation comes entirely from separate directories + separate sessions, not from any logic inside the phases themselves.

## Browser tabs: one per task, don't reuse across tasks

[policies/browser.md](./browser.md)'s "reuse the existing tab" guidance is scoped to a single task's own session — reusing *its own* ticket tab or *its own* app-under-test tab across steps of the same task. When multiple tasks are running in parallel (separate sessions per above), each task's session opens and keeps its own dedicated tab(s); don't call `tabs_context_mcp` and reuse a tab that belongs to a different task's in-progress session, since navigating it away would pull the rug out from under that other task. If it's ambiguous whose tab a given open tab belongs to, open a new one instead of guessing.

## Shared dev DB stays shared

[policies/database.md](./database.md)'s restriction is unchanged by parallelism: the test suite still must never touch the shared `development` DB, worktree or not. If [phases/verify](./verify.md) in two parallel worktrees both exercise the same shared dev DB through their (separate) dev servers, treat that the same as two people manually testing against it at the same time — fine for read/incidental use, but don't run destructive/seed operations from either without checking the other isn't mid-verification.
