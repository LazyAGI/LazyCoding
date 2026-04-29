# A. Task Kickoff

> Read this module **before starting any task**. It ensures requirements are clear, design is
> agreed upon, and work is broken into testable units.

---

## A.1 Identify the Task Type

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

## A.2 Requirement Confirmation (mandatory — do not skip)

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

## A.3 Technical Design (for non-trivial tasks)

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

## A.4 Task Breakdown

Break the task into independent sub-tasks, each of which:
- Has a clear completion criterion
- Can be tested independently
- Covers no more than one logical unit (one function, one class, one module)

**Refactoring special rule**: confirm that existing tests correctly describe the behavior of
the code being changed before touching anything. Never modify tests just to make them pass.
