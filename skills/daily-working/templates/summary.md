# Template: Chat Summary

Report this back to the user at the end of [phases/close](../phases/close.md), before asking for confirmation to write to the task tracker or open the PR:

```
Task: <id> — <subject> ({task_tracker})
Impact assessment: {low/high, one-line reason — from phases/assess.md}
Implemented: {files changed}
Tests: {pass/fail, coverage}
Browser check: {what was exercised, what was observed}
Commit: {commit message used}
```

Even a low-impact task gets the one-line impact-assessment note — the point is that the human reviewing later sees the assessment happened, not just its silence.
