# High-Quality Code Development Skill

> This skill applies to all coding tasks: new features, bug fixes, refactoring, performance
> optimization, and technical debt. It is organized into three modules:
> **[A] Task Kickoff**, **[B] Coding Standards**, **[C] Pre-Commit Checklist**.
> Each module defines clear intermediate artifacts to keep the process traceable and verifiable.

---

## Index

| Module | Content | When to Use |
|--------|---------|-------------|
| [A. Task Kickoff](#a-task-kickoff) | Requirement confirmation, design, task breakdown | Before starting any task |
| [B. Coding Standards](#b-coding-standards) | Core code quality rules (including review checklist items) | During coding |
| [C. Pre-Commit Checklist](#c-pre-commit-checklist) | Self-review, testing, doc sync | Before committing |

---

## A. Task Kickoff

### A.1 Identify the Task Type

Before writing any code, identify the task type:

| Type | Criteria | Key Risk |
|------|----------|----------|
| New Feature | Adds user-visible capability | Misunderstood requirements, incompatible interface design |
| Bug Fix | Corrects known incorrect behavior | Fix introduces new bugs, no regression test |
| Refactor | Changes code structure without changing behavior | Breaks existing functionality, tests not updated |
| Enhancement | Extends or improves existing functionality | Coupling with existing logic, compatibility breakage |
| Performance | Improves speed / memory / concurrency | Premature optimization, added complexity |
| Tech Debt | Cleans up redundant or non-standard code | Scope creep, destabilizes system |

---

### A.2 Requirement Confirmation (mandatory — do not skip)

**Before writing any code**, confirm the following with the user and output in this format:

```
## Task Confirmation

- Task type:        [New Feature / Bug Fix / Refactor / Enhancement / Performance / Tech Debt]
- Goal:             [One sentence describing what needs to be done]
- Scope:            [Which files / modules / interfaces will be modified]
- Out of scope:     [What is explicitly excluded — prevents scope creep]
- Acceptance criteria: [How to determine the task is done and correct]
- Prerequisites:    [Dependencies on other PRs / branches / external services]
```

> **If the user has not provided enough information, ask first — do not assume.**
> Especially "Scope" and "Acceptance criteria" — do not start coding until these are clear.

---

### A.3 Technical Design (for non-trivial tasks)

For tasks that touch more than 2 files or involve interface changes, output a design summary before coding:

```
## Design Summary

- Approach:         [2-3 sentences describing the implementation strategy]
- Interface changes: [Public functions / classes / parameters added, modified, or removed]
- Data flow:        [How data flows, and how it differs from the current flow]
- Risk areas:       [Where things could go wrong]
- Alternatives:     [Approaches considered but rejected, and why]
```

> Confirm the design before coding. If the user disagrees with the approach, align first.

---

### A.4 Task Breakdown

Break the task into independent sub-tasks, each of which:
- Has a clear completion criterion
- Can be tested independently
- Covers no more than one logical unit (one function, one class, one module)

**Refactoring special rule**: confirm that existing tests correctly describe the behavior of
the code being changed before touching anything. Never modify tests just to make them pass.

---

## B. Coding Standards

> These rules apply continuously during coding. The AI review tool checks all of these items —
> following them upfront reduces review round-trips.

---

### B.1 Defensive Programming: Less Is More

**Core principle: only add protection for problems that have already occurred, not for problems
that might occur.**

- Do not add `isinstance` checks, null guards, or broad `try...except Exception` blocks,
  especially for conditions already guaranteed by documentation.
- Exception catches must be specific: use `OSError`, `ValueError`, etc. Be aware of inheritance:
  `UnicodeDecodeError` is a subclass of `OSError` — do not list both (flake8 B014).
- When a contract is violated, fail fast with a clear error rather than silently swallowing
  the exception and continuing.
- Do not write dead code in `except` blocks (e.g. `if some_var:` when that variable is
  necessarily empty on the exception path).
- A silent `except: pass` must at minimum log `LOG.debug(...)` so the error is observable.

**Legitimate exceptions** — the following always require protection:
- User input (SQL injection, path traversal, XSS)
- External API responses (structure is untrusted)
- Concurrent shared state (race conditions)
- Resource lifecycle (files / connections / locks must be released on all paths)

---

### B.2 Concurrency Safety

- Objects shared across threads (tool sets, caches, counters) must be constructed independently
  inside each thread's closure — do not share mutable state across threads.
- When using `concurrent.futures`, alias `TimeoutError` as `FuturesTimeoutError` to avoid
  confusion with the built-in `TimeoutError`.
- The `timeout` argument to `as_completed(futs, timeout=N)` should reflect the expected
  duration of a **single task** — do not multiply by the number of workers; workers run in
  parallel, not serially.
- Class-level shared data structures should be initialized at class definition time, not lazily,
  to avoid TOCTOU races.
- Keep lock granularity small; never perform I/O or LLM calls while holding a lock.

---

### B.3 Function and Interface Design

- **Single responsibility**: each function does one thing. Extract helper functions when
  cyclomatic complexity exceeds the project limit.
- **Semantically distinct return values**: use a sentinel object (`_SKIP = object()`) to
  distinguish "skip this item" from "return an empty value" — do not use `return None` for both.
- **Avoid repeated calls**: do not call the same function twice in the same conditional
  (e.g. `ckpt.get('key') is not None` followed by `ckpt.get('key') or []`); cache to a variable.
- **Default arguments**: should reflect the most general call site, not be hardcoded to one
  specific caller's name.
- **Magic numbers**: extract as named constants with a comment explaining the design intent.
- **Interface changes**: when modifying a public function signature, update all call sites
  atomically — never leave the codebase in a state where only some callers are updated.

---

### B.4 Naming and Semantics

- Method names should be self-explanatory and concise. Do not redundantly include the class
  name in the method name (`ClassA.get_classA_instance()` → `ClassA.get_instance()`).
- Sibling classes (same base class or same role) should use consistent parameter names for
  analogous methods.
- When overloading dunder methods (`__or__`, `__getitem__`, etc.), the semantics must match
  mainstream conventions (bash pipe `|`, Python slice `[]`); misleading semantics must be flagged.
- Variable names should reflect content, not type (`active_users` is better than `user_list`).

---

### B.5 Code Cleanliness

- **Remove dead code**: functions / constants / mapping entries with no callers, branches
  commented "reserved for future use" with no actual usage.
- **Remove redundant exception types** (flake8 B014): do not list a subclass alongside its
  parent in the same `except` tuple.
- **Comments explain "why", not "what"**: the code itself should make the "what" obvious.
- **Avoid duplication**: the same logic appearing twice should be extracted. When extracting,
  do not over-abstract to the point of reducing readability.

---

### B.6 Architecture and Design

- **Single abstraction principle**: if the system already has one implementation of a concept,
  adding a second requires introducing a shared base class / Protocol / ABC that both
  implementations conform to.
- **Justify every change**: every modification must be able to answer "why does this need to
  change here?" Avoid unnecessary refactoring, over-engineering, or changes that violate
  framework conventions.
- **Module coupling**: when adding a feature, assess whether it increases coupling between
  modules. Prefer interfaces / events / dependency injection over direct references.
- **Testability**: new code should be easy to test. Avoid global state, hardcoded dependencies,
  and external calls that cannot be mocked.
- **Performance**: do not sacrifice readability for performance without profiling data to justify
  it. Choose appropriate data structures (`set` for membership, `dict` for mapping,
  `deque` for queues).

---

### B.7 Error Handling and Observability

- Error messages must include enough context for the caller to locate the problem (file name,
  line number, argument values).
- Distinguish between expected errors (bad user input, network timeout — use `LOG.warning`)
  and programming errors (assertion failures, type errors — use `LOG.error` or `raise`).
- Do not swallow an exception at a low level and then print a vague message at a high level.
  Either handle and log at the point of failure, or propagate upward and let the caller decide.
- Critical operations (LLM calls, file writes, network requests) must have timeout protection
  with a clear log message on timeout.

---

### B.8 Security

- **SQL injection**: use parameterized queries; never concatenate SQL strings.
- **Path traversal**: normalize user-supplied paths with `os.path.realpath` and verify they
  are within the allowed directory before use.
- **Hardcoded secrets**: keys, tokens, and passwords must not appear in code; use environment
  variables or config files.
- **Unsafe deserialization**: do not use `pickle`, `eval`, or `exec` on untrusted input.
- **Dependencies**: when adding a new dependency, check whether it is truly required; optional
  dependencies should be imported inside `try/except ImportError`.

---

## C. Pre-Commit Checklist

### C.1 Self-Review Checklist

Confirm each item before committing:

```
## Pre-Commit Self-Review

Code quality:
[ ] No broad except Exception (unless explicitly justified)
[ ] No dead code (unused functions, constants, imports)
[ ] No magic numbers (extracted as named constants)
[ ] No duplicated logic (extracted into shared helpers)
[ ] Comments explain "why", not "what"

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

### C.2 Task-Type-Specific Checks

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

### C.3 Review Diff Baseline

- Always diff against the target branch (usually `main`), not the previous commit.
- Only review changes actually introduced by the current PR; do not report pre-existing issues
  on unmodified lines.
- If a pre-existing issue is on a modified path and trivial to fix, fix it in passing; if
  complex, record it as tech debt but do not block the current PR.

---

### C.4 Version Control

- Each commit addresses one specific concern; do not mix unrelated changes.
- Commit message format: `type(scope): short description`
  where `type` is one of `feat` / `fix` / `refactor` / `perf` / `docs` / `test` / `chore`.
- Run lint before committing (`make lint-only-diff` or equivalent) and ensure no new lint
  errors are introduced.

---

*Goal: every commit passes the core review checklist on the first round, minimizing back-and-forth.*
