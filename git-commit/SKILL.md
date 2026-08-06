---
name: git-commit
description: Commit the files changed by the last completed task as a single, clean git commit. Use this whenever the user asks to commit their work, stage changes, or save the result of a task ("commit this", "commit the changes", "make a commit"). It stages only task-related files — never secrets, build artifacts, or unrelated drift — and writes a message matching the repo's tone.
---

# Git Commit

Create one well-formed commit containing exactly the work from the last task, nothing else. The point of the discipline is a clean, reviewable history: a commit that sweeps in unrelated files or secrets is the single fastest way to corrupt a repo's trustworthiness.

## Steps

### 1. Enumerate changed files

```bash
git status
git diff --stat
```

Identify the files that belong to the last task: source files, test files, `README.md`, `AGENTS.md`, `.opencode/**`, and `docs/` changes (if any).

### 2. Stage only those files

```bash
git add <path>
```

Stage each file explicitly. **Never use `git add -A` or `git add .`** — doing so risks sweeping in:

- Unrelated files
- Secrets (`.env.local`, `.env`)
- Build artifacts (`.venv/`, `__pycache__/`, `.pytest_cache/`, `target/`, `node_modules/`)
- Data files (`data/*.parquet`, `*.svg`)

If any path is gitignored, skip it silently.

### 3. Verify the staged set

```bash
git status
```

Confirm the staged files match the task and nothing else before writing anything.

### 4. Write the commit message

Inspect recent commits for tone:

```bash
git log --oneline -10
```

Format:
- **Subject**: imperative, present tense, ≤72 chars, no trailing period
- **Body**: a short WHY summary (one or two lines) and a bullet list of files changed

```bash
git commit -m "<subject>" -m "<body>"
```

### 5. Confirm

```bash
git log -1 --stat
```

## Error handling

If a pre-commit hook rejects the commit, read the error, fix the issue, and create a new commit — do not amend the failed one.
