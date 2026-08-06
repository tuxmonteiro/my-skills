---
name: add-task
description: Create a GitHub issue from a task description, silently fixing grammar and formatting. Use this whenever the user wants to add a task, open an issue, or record work to be done through GitHub issues — even if they phrase it casually ("add a task", "create an issue for X", "make a ticket for Y"). It ensures the issue title follows the project's conventions so downstream plan/execute skills work correctly.
---

# Add GitHub Task

Turn a raw task description into a well-formed GitHub issue. The user's words are the source of truth for *what* to do, but the issue title and body need to be clean enough to become a reliable entry point for the rest of the workflow (planning, execution, changelog).

## Clean the description

Take the task text and fix it silently — no commentary about the edits:

1. Correct grammar, spelling, and punctuation.
2. Capitalize the first letter of the description.
3. End with a single period, no trailing whitespace.
4. Use imperative phrasing that matches existing tasks ("Add...", "Implement...", "Fix...", "Create...").
5. Preserve exactly anything technical: file paths, CLI flags, code identifiers, formulas, library names, product names.
6. Keep the title on one line — no newlines.
7. Do not add scope the user didn't mention.
8. If the intent is ambiguous, ask for clarification instead of inventing details.

## Write a body

A title alone is enough for simple tasks whose intent is fully captured. For complex or context-heavy tasks, expand the cleaned description into a short body explaining the goal, constraints, or technical layout — the added detail helps `create-github-plan` reason about the work later.

## Execute

### 1. Sync main

```bash
git checkout main && git pull --rebase origin main
```

ABORT if the pull fails for any reason (conflicts, network error). A stale branch would create an issue against outdated context and confuse the plan workflow.

### 2. Create the issue

```bash
gh issue create -t "<cleaned description>" -b "<enhanced body or empty string>"
```

ABORT ON ERROR: if `gh` returns a non-zero exit, stop immediately and never retry — retrying can produce duplicate issues.

## Examples

| Input | Created issue title |
|-------|-------------------|
| `add a task fix typo in readme` | `Fix typo in readme.` |
| `add task implement OKX websocket client for LOB and trades` | `Implement OKX WebSocket client for LOB and trades.` |
| `add task order management system with OKX REST api integration` | `Add order management system with OKX REST API integration.` |
