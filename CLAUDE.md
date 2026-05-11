# Engineering Agent Guide

## Project Info

- **Project:** agent-core (codename: Ralph)
- **Stack:** TypeScript / Node.js / ESM / SQLite (better-sqlite3)
- **Build:** `npm run build` (tsc)
- **Test:** `npm test`
- **Lint:** `npm run lint`
- **Typecheck:** `npx tsc --noEmit`

---

## What This Project Is

`agent-core` is an autonomous code generation pipeline. It takes a spec file + a target repo, runs through Context Loader -> Planner -> Coder -> Tester <-> Fixer -> Shipper phases using configurable models via OpenRouter, and outputs a branch + PR.

We are building **Tier 1** first: a local CLI (`ralph run`, `ralph plan`) that proves the core loop works against real repos (starting with `flex-midi`). Tier 2 (Jira, Neotoma, VPS, chat bot) comes after the loop reliably produces mergeable PRs.

### Key Design Decisions

- **One model for Tier 1:** Sonnet 4.6 for all phases. We're testing the pipeline, not the model mix.
- **No Reviewer phase in Tier 1.** Prove the loop produces working code first.
- **Structured contracts between phases:** PlanStep[], FileEdit[], ReviewIssue[]. If it doesn't parse, the phase failed.
- **Never lose work:** If the loop can't finish, ship whatever exists as a draft PR.
- **Crash recovery via SQLite:** `task_state` + `run_log` tables. Resume from last completed phase on restart.
- **Record/replay for testing:** OpenRouter client supports `live | record | replay` modes. Integration tests use replay.

### Tier 1 Success Criteria

- Mergeable PR from a spec on `flex-midi` without human intervention >= 50% of the time
- Draft PR rate < 30%
- Average cost per successful task < $5
- At least 20 tasks run through it

---

## Session Initialization

At the start of every session, read these files if they exist:

1. `CONTEXT.md` -- domain model, patterns, constraints. If missing or still a blank template, offer to generate it using the `context-generation` skill.
2. `docs/lessons-learned.md` -- past mistakes; scan for entries matching files you'll touch
3. `docs/tech-debt-register.md` -- no-go zones, blockers, known risks

Read on demand (when the task touches these areas):
- `docs/superpowers/specs/2026-05-09-agent-core-design.md` -- full design spec (Tier 1 + Tier 2)

---

## Rule 1: Ask Before You Build

Never implement without explicit permission. Present options, recommend one, wait for approval.

**Exceptions:** user says "do it" / "go ahead", user asks for something specific, or you're fixing obvious typos.

---

## Rule 2: Research, Plan, Implement

```
1. RESEARCH  ->  Read docs/ first, then explore the codebase
2. PLAN      ->  Create a detailed plan, verify with the user
3. IMPLEMENT ->  TDD: failing test first, then code to pass, refactor, repeat
```

For complex tasks, follow the **pre-implementation-protocol** skill (Phase 0-5).

---

## Rule 3: Tests First (TDD)

No production code without a failing test first. Not optional.

1. Write a failing test for the expected behavior
2. Write minimal code to make it pass
3. Refactor while keeping tests green

---

## Rule 4: Keep It Simple

Simple solution wins. Don't over-engineer. If it feels complicated, stop and ask.

---

## Rule 5: Clean Code Only

**Never:** hardcode secrets, leave old code next to new, create `handleClickV2` names, leave TODOs, scatter .md files outside `docs/`.

**Always:** delete old code when replacing, use meaningful names, use early returns, keep methods short.

### Size Limits

| Element | Limit |
|---------|-------|
| Controller method | ~20 lines |
| Service class | max 10 public methods, <700 lines |
| Method | max 40-50 lines |
| Nesting depth | max 2 levels |

---

## Rule 6: "Done" Means Done

All of these must be true:

- [ ] All tests pass (run them, don't assume)
- [ ] Build succeeds (run it, don't assume)
- [ ] Old/unused code is deleted
- [ ] No errors, warnings, or lint violations

**Evidence-based verification:** Run tests (show output), run build, typecheck. "It should work" is not evidence.

If tests or build fail: STOP, fix, re-run, then resume.

---

## Rule 7: When Stuck, Ask

Stop. Present your options. Ask which direction to go.

---

## Project Architecture

### Tier 1 Repo Structure

```
agent-core/
  package.json
  tsconfig.json
  CLAUDE.md
  src/
    cli.ts                    # CLI entry point (ralph run, ralph plan)
    orchestrator.ts           # sequential phase runner with recovery
    context.ts                # TaskContext types and helpers
    config.ts                 # Zod schema, config loader
    db.ts                     # SQLite task_state + run_log
    router/
      client.ts               # OpenRouter wrapper with record/replay
    phases/
      context-loader.ts       # assemble repo context bundle
      planner.ts
      coder.ts
      tester.ts
      fixer.ts
      shipper.ts
    prompts/
      planner.md
      coder.md
      fixer.md
  tests/
    fixtures/                 # recorded OpenRouter responses
    phases/                   # unit tests per phase with stubs
    orchestrator.test.ts      # integration tests using replay mode
  docs/
    superpowers/
      specs/
```

### Key Interfaces

These are the contracts between phases. Respect them exactly.

- **TaskContext** -- accumulates state across phases (see design spec)
- **PlanStep** -- `{ order, description, files[], dependencies[], testStrategy }`
- **FileEdit** -- `{ path, action, content?, edits?: { search, replace }[] }`
- **ReviewIssue** -- Tier 2 only: `{ file, line?, severity, category, description }`

### Key Libraries

- **Zod** -- config validation
- **pino** -- structured JSON logging (every log line includes `phase`, `taskId`, `specPath`)
- **better-sqlite3** -- crash recovery + run log
- **simple-git** -- git operations
- **@octokit/rest** -- GitHub API (PR creation)

---

## Git Workflow

Check `CLAUDE.local.md` for git mode (read-only, supervised, or autonomous).

**Branches:** `feature/`, `fix/`, `hotfix/`, `design/`, `refactor/`, `chore/`, `release/` (never touch release branches unless asked).

**Commits:** atomic, clear messages explaining *why*. Never batch.

---

## Context Management

Break large tasks into focused pieces. When context gets heavy, suggest wrapping up and starting fresh. Re-read this file when context is long.

---

## Subagents

Use subagents for parallel work: exploring codebase, writing tests, delegating research, splitting refactors.

---

## Task Tracking

**Primary:** Beads (`bd`). **Fallback:** TodoWrite.

---

## Documentation

All docs go in `docs/`. No exceptions. Update docs immediately after significant changes.

---

## Skills

Check `.claude/skills/` for detailed implementation patterns. Always consult the relevant skill before generating code. When a pattern is used 3+ times, create a new skill.

### Rule Management
All rule changes go to project config files (CLAUDE.md or skills). Never store rules only in memory.

---

## Recovery Protocol

On failure: note current task, fix completely, verify the fix, return to original task.
