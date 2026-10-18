---
name: validation-evidence
description: Run this repository's checks and turn the real output into the evidence block a pull request body needs. Use before opening or updating a pull request, or whenever a change claims validation passed.
---

# Validation evidence

Follow [Contributing](../../../CONTRIBUTING.md#validate-your-change): stage new files, run the
full check set, inspect failures, and report the actual source, commands, exit results and
counts. [Taskfile](../../../Taskfile.yml) owns commands;
[Compatibility](../../../docs/compatibility.md) owns evidence limits.

A pull request body's Validation section names the checks and their results. The shape, with
every count taken from your run:

```text
`task validate` — passed, exit <exit status>.

- Markdown — <n> files, <n> errors
- link validation — <n> links, <n> errors
- consumer example contract check — <n> checks, <n> failures
- OpenTofu module contract tests — <n> passed, <n> failed
- Ansible role contract — <n> checks, <n> failures
- publication-safety patterns — <n> checks, <n> failures
- secret scan — no leaks found
- `git diff --check` — passed
- complete diff reviewed
```

The counts are deliberately not filled in: a worked example would be a set of numbers to copy.

Never copy counts from an earlier run or present source/mocked checks as live behavior. Refresh
evidence when a push changes checks or their coverage. A network retry that passes is reported
as a retry, with the affected URL; unrelated failures are reported with evidence rather than
silently fixed.
