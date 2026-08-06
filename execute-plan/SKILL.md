---
name: execute-plan
description: Execute the plan stored in a GitHub issue step by step, then post a changelog comment and open a PR. Use this whenever the user asks to execute a plan, work an issue, or run the tasks in a PLAN ("execute the plan", "work on the issue", "run the plan"). It reads the latest # PLAN from the issue (never from docs/PLAN.md), works on an exclusive git worktree branch, and finishes with a PR for human review. If more than one open issue exists, it stops and asks which one to execute.
---

# When to Use

**Always:**
- When user ask to execute the plan, execute plan, execute last plan, or execute plan #xx (when define the issue with the plan)

**Exceptions (ask your human partner):**
- All issues is closed

# Execute GitHub Plan

Execute the plan stored in a GitHub issue, end to end, leaving behind a reviewable PR. The **GitHub issue is the single source of truth** — `docs/PLAN.md` is only a temporary cache. All work happens on an exclusive worktree branch so `main` stays clean.


## Preparation

Before doing anything: check the current branch. 
- If it is not `main`, ABORT. 
- If main branch not clean (uncommitted changes, untracked files, tracked files removed but without committed the deletion), ABORT — the worktree flow below assumes a clean main.

## Steps

### 1. Create an exclusive worktree

**Use skill "using-git-worktrees"**

### 2. Download the plan from the issue

ALWAYS fetch the plan from the GitHub issue *before* reading it, saving to `docs/PLAN.md` and overriding any old version. This file is a temp/cache, never the source of truth.

```bash
ISSUE=$(gh issue list --state open --json number -L 1 -q '.[0].number')
FULL=$(gh issue view "$ISSUE" --json title,body,comments)
```

If there is more than one open issue, STOP and ask the user which issue to execute. Read only the issue the user indicates, ignoring the others.

### 3. Review the plan for conflicts

Check whether any new changes on `main` affect the plan (e.g., files the plan modifies were changed upstream). If so:

1. Identify what changed in the planned files: `git diff HEAD@{1} -- <file>`
2. Post a new issue comment with the FULL revised plan

### 4. Find the most recent plan

Scan all comments for the most recent one containing a `# PLAN` header and use it as the active plan. If no comment has a plan, fall back to the issue body.

```bash
PLAN=$(echo "$FULL" | jq -r '[.comments[] | select(.body | startswith("# PLAN"))] | last | .body // .body')
```

Save it to `docs/PLAN.md`. If the plan is outdated or missing, abort and run `create-plan` first. Extract all task sections (lines starting with `### `) and their `- [ ]` sub-steps.

### 5. Execute each task section in order

For each section:

1. Read the issue and **Subtasks** bullets to understand the problems
2. Execute each `[ ]` sub-step in order, applying the intended fix to the listed files
3. After the section completes, update the issue body with `[x]` for that section's sub-steps:
   ```bash
   gh issue edit "$ISSUE" -b "$(echo "$PLAN" | sed 's/- \[ \]/- [x]/')"
   ```

**On error** — if a sub-step fails (test, lint, command error):
- Report the failure and what caused it
- Understand the error and fix it
- Do not continue to subsequent steps
- Do not close the issue

**Progress** — the plan can have many sub-steps. Print progress feedback at least every 10 seconds so long executions stay visible.

### 6. Update `README.md` and `AGENTS.md`

If the execution changed architecture, CLI flags, added/removed commands, or changed behavior, update `README.md` and `AGENTS.md` to reflect it. Use `docs/` for all documentation and ADR references.

### 7. Post the changelog as an issue comment

The changelog summarizes what was done. It must **not** contain a `# PLAN` section.

```bash
gh issue comment "$ISSUE" -b "# Changelog

Date: $(date -u +%Y-%m-%d)
Task: <issue title>

## Summary

<what changed and why it matters>

## Files modified

- <path> — <what changed>

## Test results

<count> passed, <count> failed — <notes, e.g., coverage percentage or any regressions>."
```

### 8. Mark all sub-steps complete

All `[ ]` sub-steps should now be `[x]`. If the plan lives in the issue body, edit it:

```bash
gh issue edit "$ISSUE" -b "$(echo "$PLAN" | sed 's/- \[ \]/- [x]/g')"
```

If the plan lives in a comment, post a new comment with the completed plan.

### 9. Delete `docs/PLAN.md`

```bash
rm docs/PLAN.md
```

### 10. Create an ADR

Load `adr-writer` skill and follow these instructions.

### 11. Create a PR from this exclusive branch

1. PR title is the same as the issue title.
2. PR body is a PLAN summary, explaining what and how it fixes the issue.
3. Reference the issue in the PR body — do **not** append `🤖 Generated with [Claude Code](https://claude.com/claude-code)` or any other auto-attribution line.
4. Commit the work using the `git-commit` skill, in this branch.

Do not merge the PR — it will be checked by a human or another agent.

### 12. Return to main branch and remove the worktree

```bash
cd $PROJECT_ROOT
git worktree remove $WORKTREE
```

### 13. Show PR link

Show the full link to PR

## Full workflow

```
check if in main branch → fetch remote main and create worktree → read issue → find latest plan → review plan → execute plan → update README + AGENTS.md → post changelog → update issue body/comments → delete PLAN.md → create ADR → create PR → return to main and remove worktree →  Show PR link
```
