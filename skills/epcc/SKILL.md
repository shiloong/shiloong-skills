---
name: epcc
description: Explore-Plan-Code-Commit structured development workflow. Guides tasks through four phases with explicit verification checkpoints. Use when the user invokes /epcc or asks to implement a feature using the EPCC process.
version: 0.1.0
---

# EPCC — Explore, Plan, Code, Commit

A structured four-phase workflow for implementing features, fixing bugs, or making non-trivial changes. Each phase has a verification checkpoint before proceeding.

## Trigger

- `/epcc <task description>`
- "用 EPCC 流程实现..."
- "按 EPCC 做这个任务"

## Phase 1: Explore

**Goal**: Understand the relevant codebase before writing anything.

### Steps

1. Identify which component the task belongs to (use CLAUDE.md scope mapping)
2. Use `@file_path` to point at specific files — do NOT broad-search the entire repo
3. Read the files that will be affected, understand call chains and dependencies
4. Check git history for recent changes in those files: `git log --oneline -10 <file>`
5. Identify existing utilities, patterns, and conventions to reuse

### Verify checkpoint

Before moving to Plan, confirm:
- [ ] You can name every file that will likely be touched
- [ ] You understand the call chain / dependency graph for the change
- [ ] You know what existing patterns to follow (not invent new ones)

**Report**: List affected files, dependencies, and patterns found. Ask user to confirm before proceeding.

## Phase 2: Plan

**Goal**: Design the implementation approach. NO code changes in this phase.

### Steps

1. Enter Plan mode (Shift+Tab ×2 or `/plan`) if not already in it
2. Write a concrete implementation plan:
   - Files to create / modify / delete
   - What changes in each file (function-level, not line-level)
   - Order of implementation steps
   - Potential risks and edge cases
   - How to verify each step
3. Front-load constraints: put the most important restrictions first (e.g., "must not change config files", "only modify this one function")

### Verify checkpoint

Before moving to Code, confirm:
- [ ] Every change step has a verification method (test, build, behavior check)
- [ ] No step introduces changes beyond what the task requires
- [ ] Risk cases are identified with mitigation plans

**Report**: Present the full plan. Wait for user approval. User can modify (delete steps, add constraints, reorder) before approval.

## Phase 3: Code

**Goal**: Implement the plan step by step, verifying after each step.

### Steps

1. Exit Plan mode (Shift+Tab ×2 back to Act mode, or user approves)
2. Implement changes in order from the plan
3. After each meaningful change, run the relevant CI checks:
   - Rust: `cargo fmt --check && cargo clippy -- -D warnings && cargo test`
   - TypeScript: `npm run lint:ci && npm run build && npm run typecheck && npm run test:ci`
   - Python: `ruff check && pytest`
4. If verification fails, fix the issue before proceeding to the next step
5. Do NOT accumulate unverified changes — fix-then-continue, not fix-at-end

### Anti-patterns to avoid

- ❌ Making changes across 5 files then running tests once at the end
- ❌ Skipping verification because "it looks right"
- ❌ Adding unrelated improvements while implementing the task
- ❌ Continuing to patch when stuck in a loop — stop and report instead

### Verify checkpoint

Before moving to Commit, confirm:
- [ ] All planned changes are implemented
- [ ] All CI checks pass
- [ ] No unintended side effects introduced
- [ ] Tests cover the new behavior (or existing tests validate it)

## Phase 4: Commit

**Goal**: Create a clean, well-described commit.

### Steps

1. Review the final diff: `git diff` — verify every line traces back to the original task
2. Stage only the files involved in the task (not unrelated changes)
3. Write a commit message following the project's format (see CLAUDE.md forced constraints):
   - Format: `type(scope): description`
   - English, lowercase start, no trailing period
   - Scope maps to component path (see CLAUDE.md scope→path table)
4. Add SOB: `Signed-off-by: Shile Zhang <shile.zhang@linux.alibaba.com>` (only this signer, no others)
5. Do NOT push — that requires explicit user action per CLAUDE.md rules

### Verify checkpoint

- [ ] Commit message follows `type(scope): description` format
- [ ] Scope is correct per CLAUDE.md mapping
- [ ] SOB line is present and correct
- [ ] Diff contains only task-related changes (zero unrelated lines)
- [ ] All CI pre-commit checks would pass

## Key Principles

1. **Verification is the highest leverage** — every phase has a checkpoint
2. **One task, one session** — don't mix unrelated work
3. **Plan before code** —纠偏成本在 Plan phase 最低
4. **Fix-then-continue** — don't accumulate unverified changes
5. **Minimal diff** — every line in the final diff must trace back to the user's original request