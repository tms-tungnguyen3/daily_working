# Testing

Part of [phases/implement](./implement.md) (step 2). Write/adjust tests using `tests.run_command` and `tests.coverage_target` from `.claude/daily-working.yml` — or infer them from the project if unset. Put specs alongside the existing spec files for the module you touched, following that module's existing spec structure/factories rather than starting a new one.

## ⚠️ Database safety — read before running any spec

See [policies/database](../policies/database.md) for the full rule. In short: `development` is commonly a **shared** DB other people/environments depend on, `test` is yours to use freely — always run with the test environment explicit, verify the resolved DB isn't the shared host before the first run in a new shell, and never let a spec run touch `development` (no migrations, seeds, console mutations, ad-hoc scripts). This does not apply to [phases/verify](./verify.md) — browser-driven E2E against the running dev server is expected and fine.
