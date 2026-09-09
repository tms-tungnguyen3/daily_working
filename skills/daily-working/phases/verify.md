# Phase 3: Verify via Claude Chrome Extension

Part of [workflows/implement](../workflows/implement.md) (and the primary step of [workflows/review](../workflows/review.md)). Runs after [phases/implement](./implement.md) — or, in [workflows/review](../workflows/review.md), against code that already exists.

Use `claude-in-chrome` to drive the actual running app instead of relying solely on the test suite/system specs.

1. Confirm the dev server is running (use `dev_server.default_url` from the config file, or ask the user for the local URL if unset)
2. Load browser tools per [policies/browser](../policies/browser.md)
3. Call `tabs_context_mcp` first, then open/reuse a tab and navigate to the page affected by the task
4. Exercise the exact flow described in the ticket (click through, fill forms, trigger the changed behavior)
5. Check for: correct rendered output, no console errors, expected network responses, and that the original bug/request from the tracker is actually resolved
6. If something's off, go back to [phases/implement](./implement.md) — don't report success on stubbed/assumed behavior
