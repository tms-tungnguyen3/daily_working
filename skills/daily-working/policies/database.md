# Policy: Database Safety

⚠️ Read before running any spec in [phases/test](../phases/test.md).

Check `database_safety.shared_dev_db` in `.claude/daily-working.yml` first.

`development` in a project's DB config commonly points at a **shared** dev DB that multiple people/environments depend on staying intact, while `test` points at a local/isolated DB that's yours to use freely — reset, seed, wipe, whatever. If `database_safety.shared_dev_db` is `true`, treat that as confirmed for this project and also follow `database_safety.notes`. If it's `false`/unset, this restriction may not apply here — but still verify which DB the test environment resolves to before the first run in a new shell/session, since defaults vary per project.

This restriction is specifically about **running the test suite** — it must never connect to the shared dev DB:

1. **Always run with the test environment explicit** — e.g. `RAILS_ENV=test bundle exec rspec ...` for Rails/rspec, or the equivalent env flag for whatever stack/test runner this project uses (`tests.run_command` in the config file, if set). Don't rely on a default that might resolve to `development`.
2. **Verify the resolved DB config isn't the shared/dev host** before the first run in a new shell/session — print or eyeball the active test DB config and confirm it's the local/isolated one, not the `development` one.
3. **Never let a spec run touch the dev DB.** No migrations, seeds, console mutations, or ad-hoc scripts against `development`/the shared host via the test suite — even for "just checking something." If a check requires touching data, do it against the test DB or ask the user first.

## Does not apply to browser verification

This restriction is scoped to the test suite. It does **not** apply to [phases/verify](../phases/verify.md) — browser-driven E2E verification against the running dev server (and its dev DB) is expected and fine, no special caution needed there.
