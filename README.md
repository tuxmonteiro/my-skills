# OpenCode Skills Repository

This repository contains specialized skills for OpenCode, an interactive CLI tool for software engineering tasks. Each skill provides structured workflows and patterns for specific development scenarios.

## Skills Overview

| Skill | Purpose | When to Use |
|-------|---------|-------------|
| [add-task](#add-task) | Create well-formed GitHub issues from task descriptions | Adding tasks, opening issues, recording work |
| [adr-writer](#adr-writer) | Write Architecture Decision Records | Creating ADRs, documenting decisions, justifying approaches |
| [create-plan](#create-plan) | Turn GitHub issues into executable plans stored in the issue | Planning work, breaking down tasks into steps |
| [execute-plan](#execute-plan) | Execute plans from GitHub issues end-to-end with PR creation | Running plans, working on issues, implementing features |
| [git-commit](#git-commit) | Create clean, single-purpose git commits | Committing completed work |
| [rust-coding](#rust-coding) | Unified Rust patterns and coding standards | Writing, reviewing, or refactoring Rust code |
| [rust-tdd](#rust-tdd) | Test-Driven Development workflow for Rust | Implementing features or bugfixes in Rust |
| [skill-creator](#skill-creator) | Create, improve, and benchmark OpenCode skills | Building new skills or optimizing existing ones |
| [using-git-worktrees](#using-git-worktrees) | Set up isolated workspaces for feature work | Starting feature work needing isolation |
| [verification-before-completion](#verification-before-completion) | Enforce evidence-based completion claims | Before claiming work is done, fixed, or passing |

---

## add-task

**Description:** Create a GitHub issue from a task description, silently fixing grammar and formatting. Ensures the issue title follows project conventions so downstream plan/execute skills work correctly.

**Key features:**
- Auto-cleans task descriptions (grammar, capitalization, punctuation)
- Enforces imperative phrasing ("Add...", "Implement...", "Fix...")
- Preserves technical details (file paths, CLI flags, code identifiers)
- Syncs `main` before creating issues
- Aborts on `gh` errors to prevent duplicates

**Workflow:**
1. Clean the raw task description
2. Write enhanced body for complex tasks
3. Sync main branch
4. Create GitHub issue with `gh issue create`

**Example:**
- Input: `add a task fix typo in readme`
- Output: `Fix typo in readme.`

---

## adr-writer

**Description:** Write an ADR file. Use this whenever the user asks to create an ADR, document a decision, explain an approach, or justify why X was chosen over Y.

**Key features:**
- Creates Architecture Decision Records with structured format
- Organizes ADRs by category (Core Architecture, Integration, Persistence & Storage, Metrics & Visualization, Operations)
- Auto-generates sequential numbering from existing ADRs
- Enforces consistent file naming: `ADR-<num>-<YYYYMMDD>-<short-title>.md`
- Saves to `docs/adr/<category>/` directory structure

**When to use:**
- Always: When the user asks to create an ADR, document a decision, explain an approach, or justify why X was chosen over Y
- Exceptions (ask human): Unclear if task requires a complex decision; task is debugging, fixing an issue, or answering a simple question

**ADR Structure:**
```markdown
Title: Sequential number + active-voice decision statement
Category: See Category Mapping below
Status: Proposed | Accepted | Rejected | Superseded
Created: YYYY-MM-DD HH:MM
Context: Forces, requirements, and background circumstances
Options Considered: Serious alternatives with pros and cons
Decision: Chosen solution and brief justification
Consequences: Positive and negative implications, including trade-offs
```

**Category Mapping:**

| Category | Topics |
|----------|--------|
| Core Architecture | Foundational technology choices, framework decisions, language/runtime choices |
| Integration | Exchange-specific modules, traits, instrument resolution, URL config |
| Persistence & Storage | Database choice, schema, migrations, retention policies |
| Metrics & Visualization | Prometheus, Grafana, dashboard layout, endpoint structure |
| Operations | Deployment, networking, signaling, reliability, workflow changes |

**File naming:** `ADR-<sequential number %03d>-<YYYYMMDD>-<short-title>.md`
Example: `ADR-042-20260801-grab-your-gun-and-bring-in-the-cat.md`

---

## create-plan

**Description:** Read the latest open GitHub issue and write a `PLAN.md` with implementation sub-steps, then store it in the issue body (if empty) or as a new comment. It only plans — never executes.

**Key features:**
- Reads full issue context (title, body, comments)
- Creates `docs/PLAN.md` with files to modify, subtasks, verification commands
- Stores plan in GitHub issue (single source of truth)
- Includes ADR creation sub-tasks when needed
- Uses markdown checklists (`- [ ]`) for sub-steps
- Plan updates always replace with full plan, never partial

**Plan structure:**
```markdown
# PLAN
Task: <issue title>

## Files to modify
- <path>

## Subtasks
### 1. <header>
Issues found:
- <problem>

- [ ] <action with file paths>

## Verification
<shell commands>

## Changelog
Post changelog comment, then close issue.
```

---

## execute-plan

**Description:** Execute the plan stored in a GitHub issue step by step, then post a changelog comment and open a PR. Works on an exclusive git worktree branch, leaving `main` clean.

**Key features:**
- Requires clean `main` branch before starting
- Creates isolated worktree (uses `using-git-worktrees` skill)
- Fetches plan from GitHub issue (never from local `docs/PLAN.md`)
- Updates issue body with `[x]` as steps complete
- Runs verification after each section
- Updates `README.md` and `AGENTS.md` if architecture changes
- Posts changelog comment, creates ADR, opens PR
- Returns to `main` and removes worktree

**Full workflow:**
```
check main → create worktree → read issue → find latest plan → 
review conflicts → execute plan → update README + AGENTS.md → 
post changelog → mark complete → delete PLAN.md → create ADR → 
verify sidebar → create PR → return to main → show PR link
```

---

## git-commit

**Description:** Commit the files changed by the last completed task as a single, clean git commit. Stages only task-related files — never secrets, build artifacts, or unrelated drift.

**Key features:**
- Enumerates changed files explicitly
- Stages each file individually (never `git add -A` or `git add .`)
- Verifies staged set matches task
- Writes commit message matching repo tone
- Subject: imperative, present tense, ≤72 chars
- Body: WHY summary + bullet list of files changed
- Creates new commit on pre-commit hook rejection (no amend)

**Excluded from commits:**
- Secrets (`.env.local`, `.env`)
- Build artifacts (`.venv/`, `target/`, `node_modules/`)
- Data files (`*.parquet`, `*.svg`)
- Unrelated files

---

## rust-coding

**Description:** Unified Rust engineering skill covering idiomatic patterns (ownership, error handling, enums, traits, concurrency, unsafe code) and baseline coding standards (naming, module organization, documentation, testing, code smells). Merges `rust-patterns` and `coding-standards`.

**Core principles:**
1. **Readability First** — Code is read more than written
2. **KISS** — Prefer simplest ownership model
3. **DRY** — Extract reusable functions, prefer traits
4. **YAGNI** — Avoid speculative generics

**Key topics covered:**
- **Naming:** Variables (snake_case), Functions (snake_case), Types (PascalCase), Constants (SCREAMING_SNAKE_CASE)
- **Ownership:** Borrow over clone, `mut` only when needed, `Cow` for flexible ownership
- **Error Handling:** `Result`/`thiserror` for libraries, `anyhow` for apps, no `unwrap()` in production
- **Enums:** Make illegal states unrepresentable, exhaustive matching
- **Traits:** Small focused traits, accept generics return concrete, newtype pattern
- **Modules:** Domain-first organization, layer inside domains only when needed
- **Concurrency:** `Arc<Mutex<T>>`, channels, structured async with `tokio::join!`
- **Unsafe:** Only for FFI/performance with `// SAFETY:` comments
- **Testing:** AAA pattern, behavior-focused names (via `rust-tdd` skill)
- **Tooling:** `cargo clippy`, `cargo fmt`, `cargo audit`, `cargo bench`

**Anti-patterns to avoid:** `.unwrap()`, cloning to silence borrow checker, `String` when `&str` suffices, `Box<dyn Error>` in libraries, blocking in async.

---

## rust-tdd

**Description:** Test-Driven Development workflow for Rust. Enforces the iron law: **NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST**.

**When to use:**
- New features
- Bug fixes
- Refactoring
- Behavior changes

**Exceptions (require human permission):**
- Throwaway prototypes
- Generated code
- Configuration files

**Red-Green-Refactor Cycle:**

1. **RED** — Write failing test (one behavior, clear name, real code, no mocks unless unavoidable)
2. **Verify RED** — Run test, confirm it fails for expected reason
3. **GREEN** — Write minimal code to pass (no over-engineering)
4. **Verify GREEN** — Run test, confirm passes, other tests still pass
5. **REFACTOR** — Clean up (remove duplication, improve names, run clippy)
6. **Repeat** — Next failing test

**Red flags (STOP and start over):**
- Code before test
- Test passes immediately
- "Tests after achieve same goals"
- "Already manually tested"
- "Keep as reference, write tests first"

**Verification checklist:**
- [ ] Every new function has a test
- [ ] Watched each test fail before implementing
- [ ] Each test failed for expected reason
- [ ] Wrote minimal code to pass
- [ ] All tests pass
- [ ] Output pristine (no errors/warnings)

---

## skill-creator

**Description:** Create new skills, modify and improve existing skills, and measure skill performance. Handles the full lifecycle from intent capture to packaging.

**Core loop:**
1. **Capture Intent** — Understand what the skill should do and when to trigger
2. **Interview & Research** — Ask about edge cases, formats, dependencies
3. **Write SKILL.md** — Name, description (primary trigger), compatibility, instructions
4. **Test Cases** — 2-3 realistic prompts, save to `evals/evals.json`
5. **Run & Evaluate** — Spawn with-skill and baseline runs in parallel
6. **Grade & Benchmark** — Quantitative assertions + qualitative review
7. **Improve** — Generalize from feedback, explain why, bundle repeated scripts
8. **Repeat** — Until user satisfied or feedback empty

**Advanced features:**
- **Blind comparison** — Independent agent judges two outputs without knowing which is which
- **Description optimization** — 20 eval queries (should/should-not trigger), automated optimization loop
- **Claude.ai / Cowork adaptations** — Works without subagents or browser

**Skill anatomy:**
```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description)
│   └── Markdown instructions
└── Bundled Resources (optional)
    ├── scripts/    - Executable code
    ├── references/ - Docs loaded as needed
    └── assets/     - Templates, icons, fonts
```

---

## using-git-worktrees

**Description:** Ensure work happens in an isolated workspace INSIDE the current project root. Prefers native worktree tools and falls back to manual git worktrees.

**Core principle:** Detect existing isolation first → use native tools → fall back to git. Never create workspace in `/tmp` or outside project.

**Step 0: Detect Existing Isolation**
- Check `GIT_DIR` vs `GIT_COMMON`
- Guard against submodules
- If already in worktree: skip creation

**Step 1: Create Isolated Workspace**
1. **Native tools preferred** (e.g., `EnterWorktree`, `WorktreeCreate`)
2. **Git worktree fallback:**
   - Directory priority: explicit preference → `.worktrees/` → `worktrees/` → default `.worktrees/`
   - **MUST verify ignored** with `git check-ignore`
   - Branch format: `<tag>/<short-description>` (e.g., `feat/user-auth`)
   - Fetch and rebase from `origin/main`

**Step 2: Project Setup**
- Auto-detect: `npm install`, `cargo build`, `pip install`, `go mod download`

**Step 3: Verify Clean Baseline**
- Run project tests (`npm test`, `cargo test`, `pytest`, `go test ./...`)
- Report failures, ask whether to proceed

**Common mistakes avoided:**
- Skipping detection (harness/submodules fool eyeballing)
- Using `git worktree add` when native tool exists
- Not verifying `.worktrees/` is gitignored
- Working in unignored directory (commits worktree to repo)

---

## verification-before-completion

**Description:** Enforce evidence-based completion claims. Requires running verification commands and confirming output before making any success claims.

**Iron Law:** NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE

**The Gate Function:**
```
BEFORE claiming any status:
1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
   - NO: State actual status with evidence
   - YES: State claim WITH evidence
5. ONLY THEN: Make the claim
```

**What requires verification (not sufficient):**
| Claim | Requires | Not Sufficient |
|-------|----------|----------------|
| Tests pass | Test output: 0 failures | Previous run, "should pass" |
| Linter clean | Linter output: 0 errors | Partial check |
| Build succeeds | Build command: exit 0 | Linter passing |
| Bug fixed | Test original symptom: passes | Code changed, assumed fixed |
| Agent completed | VCS diff shows changes | Agent reports "success" |

**Red flags (STOP):**
- Using "should", "probably", "seems to"
- Expressing satisfaction before verification ("Great!", "Perfect!", "Done!")
- About to commit/PR without verification
- Trusting agent success reports
- Relying on partial verification

**Key patterns:**
```bash
# Tests
✅ cargo test → "All 34 tests pass"
❌ "Should pass now"

# Regression (TDD Red-Green)
✅ Write → Run (pass) → Revert → Run (MUST FAIL) → Restore → Run (pass)
❌ "I've written a regression test"

# Build
✅ cargo build → exit 0
❌ "Linter passed"

# Requirements
✅ Re-read plan → Checklist → Verify each
❌ "Tests pass, phase complete"
```

**Applies ALWAYS before:** any completion claim, satisfaction expression, commit/PR, moving to next task, delegating to agents.

---

## Quick Reference: Skill Combinations

| Scenario | Skills to Use |
|----------|---------------|
| New feature request | `add-task` → `create-plan` → `execute-plan` |
| Bug fix | `add-task` → `create-plan` → `execute-plan` (or direct `rust-tdd`) |
| Rust implementation | `rust-tdd` + `rust-coding` |
| Starting feature work | `using-git-worktrees` |
| Committing work | `git-commit` |
| Before declaring done | `verification-before-completion` |
| Creating new skill | `skill-creator` |
| Improving existing skill | `skill-creator` |
| Documenting architectural decisions | `adr-writer` (used by `create-plan`/`execute-plan`) |

---

## Repository Structure

```
skills/
├── add-task/
│   └── SKILL.md
├── adr-writer/
│   └── SKILL.md
├── create-plan/
│   └── SKILL.md
├── execute-plan/
│   └── SKILL.md
├── git-commit/
│   └── SKILL.md
├── rust-coding/
│   └── SKILL.md
├── rust-tdd/
│   └── SKILL.md
├── skill-creator/
│   ├── SKILL.md
│   ├── agents/
│   ├── references/
│   └── scripts/
├── using-git-worktrees/
│   └── SKILL.md
├── verification-before-completion/
│   └── SKILL.md
└── README.md
```

Each skill is self-contained in its directory with a `SKILL.md` file containing YAML frontmatter (name, description) and markdown instructions.
