# C. Pre-Commit Checklist

> Read this module **before committing**. Walk through each section to catch issues early.

---

## C.1 Self-Review Checklist

Confirm each item before committing:

```
## Pre-Commit Self-Review

Code quality:
[ ] No broad except Exception (unless explicitly justified)
[ ] No dead code (unused functions, constants, imports)
[ ] No magic numbers (extracted as named constants)
[ ] No duplicated logic (extracted into shared helpers)
[ ] Comments explain "why", not "what"

Conciseness and readability:
[ ] No deeply nested if-else (used early return where possible)
[ ] Similar conditional branches merged into a single path
[ ] Functions are ≤20 lines (complex logic extracted into helpers)
[ ] No function has more than 4 positional parameters without grouping

Concurrency safety:
[ ] Shared objects are constructed independently per thread
[ ] Timeout values are NOT multiplied by worker count

Interface and architecture:
[ ] All call sites updated when a public interface changes
[ ] A shared base class introduced when adding a second implementation
[ ] Every change can answer "why does this need to change here?"

Testing:
[ ] New / modified public functions have corresponding test cases
[ ] Refactoring did not change test semantics (tests still correctly describe behavior)
[ ] Edge cases and error paths are covered

Documentation:
[ ] Core flow / interface / return-value changes are reflected in docs
[ ] Function names and parameter names in docs match the implementation
```

---

## C.2 Task-Type-Specific Checks

**Bug Fix:**
- Does the fix change only what is necessary, with no unrelated modifications?
- Is there a regression test that covers the bug scenario?
- Is the root cause addressed, not just the symptom?

**Refactor:**
- Is the behavior identical before and after (all tests pass)?
- Are all paths touched by the refactor covered by tests?
- Does the refactor genuinely improve readability / maintainability, or is it just a different style?

**Performance Optimization:**
- Is there profiling data justifying the optimization?
- Is there benchmark data showing the improvement?
- Does the optimization introduce new complexity or maintenance burden?

**New Feature:**
- Are all acceptance criteria from the Task Confirmation covered?
- Are edge cases and error paths handled?
- Is the new feature consistent in style with the existing codebase?

---

## C.3 Review Diff Baseline

- Always diff against the target branch (usually `main`), not the previous commit.
- Only review changes actually introduced by the current PR; do not report pre-existing issues
  on unmodified lines.
- If a pre-existing issue is on a modified path and trivial to fix, fix it in passing; if
  complex, record it as tech debt but do not block the current PR.

---

## C.4 Version Control

- Each commit addresses one specific concern; do not mix unrelated changes.
- Commit message format: `type(scope): short description`
  where `type` is one of `feat` / `fix` / `refactor` / `perf` / `docs` / `test` / `chore`.
- Run lint before committing (`make lint-only-diff` or equivalent) and ensure no new lint
  errors are introduced.

---

*Goal: every commit passes the core review checklist on the first round, minimizing back-and-forth.*
