---
name: create-plan
description: Read the latest open or user indicated GitHub issue and write a PLAN.md with implementation sub-steps, then store it in the issue body (if empty) or as a new comment. Use this whenever the user asks to plan an issue, write a PLAN.md, or break down a GitHub task into steps ("create a plan", "plan this issue", "make a PLAN.md"). It only plans — it never executes the plan. Pair with execute-plan which runs it.
---

# Create Plan

Turn the latest open GitHub issue into an executable plan, stored where `execute-plan` can find it. The plan lives in the **GitHub issue itself** (body or comment); `docs/PLAN.md` is just a local working copy. The issue is the single source of truth, so the plan must always round-trip through it.

## Steps

### 1. Get the last open issue and all its context

```bash
ISSUE=$(gh issue list --state open --json number -L 1 -q '.[0].number')
gh issue view "$ISSUE" --json title,body,comments
```

Read everything: title, existing body, and previous comments. Past comments often reveal constraints or decisions that shape the plan.

### 2. Create ADR

- If the plan changes behavior or introduces a new approach (not a simple refactor or bug fix), add a subtask to **Create an ADR** and load skill `adr-writer` to define the future ADR file name and category directory.

### 3. Check if the issue body is empty

```bash
BODY=$(gh issue view "$ISSUE" --json body -q '.body')
```

If the body is empty or null, store the plan there (first plan for this issue):

```bash
gh issue edit "$ISSUE" -F docs/PLAN.md
```

If the body already has content, or this is an update to an existing plan, post the plan as a new comment instead:

```bash
gh issue comment "$ISSUE" -F docs/PLAN.md
```

**IMPORTANT — Plan updates must always be the full plan.** Every update repeats the complete plan, never just the changes. Never edit or replace a previous plan comment — always post a new one. The most recent comment containing `# PLAN` is the canonical source for `execute-plan`.

## Template

When drafting `docs/PLAN.md`, use this structure:

```markdown
# PLAN

Task: <issue title>

## Files to modify

- <path>

## Subtasks

### 1. <descriptive header>

Issues found:
- <problem in current state>

- [ ] <action with file paths>
- [ ] <next action>

### 2. ...

### N. Create ADR (if necessary. Check `adr-writer` skill conditions)

 Issues found:
 - <why an ADR is needed>

- [ ] Create <ADR file name> documenting the decision
- [ ] <Add next substeps defined in `adr-writer` skill> ...

## Verification

```
<shell commands to confirm correctness>
```

## Changelog

After execution, post a changelog as a comment on this issue (without a PLAN section).
```

## Important

- **Never execute the plan** — this skill only creates and stores it
- Sub-steps use markdown checklist format: `- [ ]`
- Verification commands must be runnable shell snippets
