# D. Large Task & Cross-Session Workflow

> Read this module when starting a task that spans multiple sessions, is tracked as a dedicated
> PR/branch, or touches more than 3 modules. It covers session tracking, persistent plan files,
> cross-session constraint memory, and end-of-PR constraint promotion.
>
> Refactoring tasks follow all rules in this module plus the additional constraints in D.7.

---

## D.1 When This Module Applies

Use this workflow when **any** of the following is true:

- The task is tracked as a dedicated PR or branch
- The session log (see D.2) shows ≥2 sessions on the current branch
- The task touches more than 3 modules or involves interface changes across the codebase
- The codebase is in a "mid-refactor" state (new and old logic coexist)

For small, single-session tasks, this module is optional.

---

## D.2 Session Tracking File

All sessions are logged in `.cursor/session-log.json` at the repo root. This file records
which sessions have worked on which branch, and is the trigger for plan file creation.

### Format

```json
{
  "feature/my-branch": [
    {
      "timestamp": "2026-05-08T10:23:00",
      "summary": "Added _Processor base class, removed algo_id from signature"
    },
    {
      "timestamp": "2026-05-08T14:05:00",
      "summary": "Updated all callers of _enqueue_task, fixed lint errors"
    }
  ]
}
```

### Rules

- **On session start**: read `.cursor/session-log.json`. Check if the current branch already
  has entries. If the branch has ≥1 existing entry, this is a multi-session task — follow D.3.
- **On session end**: append a new entry for the current branch with the current timestamp
  and a one-sentence summary of what was done.
- If the file does not exist, create it with an empty object `{}` and add the first entry.
- The `summary` field should be one sentence describing the main change made in this session.
  It is used to orient the next session quickly — write it for the next AI, not for the user.

---

## D.3 Multi-Session Task: Plan File

When a branch has ≥2 sessions in the log (i.e. the current session is the second or later),
a plan file must exist at:

```
.cursor/plans/<branch-name>.md
```

If the plan file does not exist yet, create it immediately before doing any other work,
using the session log entries as context for the "Current State" section.

### Required sections

```markdown
## Goal
[One sentence: what this PR/branch achieves when merged]

## Core Constraints
[Bullet list of design decisions that must not be violated — written as prohibitions]
- DO NOT add X to Y (reason: ...)
- X must be shared globally, not per-instance (reason: ...)
- Only use the plural form `xxxs`; the singular `xxx` has been removed

## Key Concepts / Glossary
[Define terms whose meaning is non-obvious or project-specific]
- "global": shared across all instances via a class-level registry, not per-instance

## Current State
[What has been done, what remains — updated at the end of each session]
- [x] Removed algo_id from _Processor
- [ ] Update all callers of _enqueue_task

## Out of Scope
[What is explicitly excluded from this PR]
```

### Rules for maintaining the plan file

- **Read first**: at the start of every session on this branch, read the plan file before
  writing any code.
- **Update last**: at the end of every session, update "Current State" to reflect what was done.
- **Constraints are prohibitions**: write constraints as "DO NOT" or "must not" — this is more
  reliably followed than positive goals.
- **Glossary for ambiguous terms**: if a term has a project-specific meaning that differs from
  its common meaning (e.g. "global", "singleton"), define it with a code reference or example.

---

## D.4 Mid-Task Code Is Unreliable as a Source of Truth

When a task is in progress, the codebase may contain patterns that are being removed or
replaced. **Do not infer design intent from the current code state alone.**

Always derive intent from, in priority order:
1. The plan file (highest priority)
2. Explicit user instructions in the current session
3. The session log summaries from previous sessions
4. The code state (lowest priority — may reflect old or transitional design)

If the plan file and the code state contradict each other, flag the contradiction to the user
before proceeding.

---

## D.5 Session Start Checklist

```
[ ] Read .cursor/session-log.json — check how many sessions this branch has had
[ ] If ≥2 sessions: read .cursor/plans/<branch>.md before doing anything else
[ ] Re-read "Core Constraints" — treat them as hard rules for this session
[ ] Check "Current State" to understand what has already been done
[ ] Ask the user if the plan has changed since the last session
```

---

## D.6 Session End Checklist

```
[ ] Append a new entry to .cursor/session-log.json (timestamp + one-sentence summary)
[ ] Update "Current State" in the plan file (if it exists)
[ ] Add any new constraints the user stated during this session to "Core Constraints"
[ ] Note any unresolved questions or blockers in the plan file
```

---

## D.7 Additional Rules for Refactoring Tasks

These rules apply on top of D.1–D.6 when the task type is Refactor (see A.1).

- **Confirm test validity first**: before changing any code, confirm that existing tests
  correctly describe the current behavior. A test that was written for the old design is
  outdated — update it to match the new design, do not revert the refactor to satisfy it.
- **Behavior must be identical**: all tests must pass after the refactor. If a test fails,
  determine whether the test is outdated or the refactor broke real behavior before acting.
- **No scope creep**: a refactor must not add new features or change observable behavior.
  If a bug is discovered during refactoring, record it as a separate task rather than fixing
  it in place.

---

## D.8 Constraint Promotion at PR Close

When a PR is ready to merge (tests pass, lint clean, review approved), suggest to the user
that constraints discovered during this PR be promoted to project-level rules.

**Trigger**: after a successful push that passes CI, or when the user says the PR is done.

**Action**: output a suggestion in this format:

```
## Constraint Promotion Suggestion

The following constraints were added to this PR's plan file and may be worth
promoting to the project's permanent coding standards:

1. [Constraint text] → suggested location: `coding-standards.mdc` / `AGENTS.md`
   Reason: [why this is general enough to apply beyond this PR]

2. ...

To promote: add them to `.cursor/rules/coding-standards.mdc` or the project's `AGENTS.md`.
Skip any that are specific to this PR's context and unlikely to recur.
```

Only suggest promotion for constraints that are:
- General enough to apply to future PRs
- Not already covered by existing rules
- Derived from a real mistake or ambiguity encountered during this PR (not speculative)
