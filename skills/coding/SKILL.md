# High-Quality Code Development Skill

> This skill applies to all coding tasks: new features, bug fixes, refactoring, performance
> optimization, and technical debt. It is organized into three modules, each in a separate file.
> Modules A and C are phase-specific (read when entering that phase).
> **Module B should be loaded in full** at the start of coding — it contains all quality rules
> that apply continuously.

---

## Module Index

| Module | File | When to Read |
|--------|------|-------------|
| A. Task Kickoff | [a-task-kickoff.md](./a-task-kickoff.md) | **Before starting any task** — requirement confirmation, design, task breakdown |
| B. Coding Standards | [b-coding-standards.md](./b-coding-standards.md) | **During coding** — core quality rules with Bad/Good examples |
| C. Pre-Commit Checklist | [c-pre-commit.md](./c-pre-commit.md) | **Before committing** — self-review, testing, doc sync, version control |
| D. Large Task | [d-large-task.md](./d-large-task.md) | **For multi-session / PR-scoped tasks** — session tracking, plan file, cross-session constraints, constraint promotion |

---

## How to Use

1. **Starting a task** → read `a-task-kickoff.md`. Confirm requirements and design before writing code.
2. **Writing code** → read `b-coding-standards.md` **in full**. All sections apply
   continuously throughout coding — load once, refer back as needed.
3. **Ready to commit** → read `c-pre-commit.md`. Walk through the checklist before pushing.
4. **Multi-session / PR-scoped task** → read `d-large-task.md`. Track sessions in `.cursor/session-log.json`, create a plan file when ≥2 sessions detected, and suggest constraint promotion when the PR closes.

*Goal: every commit passes the core review checklist on the first round, minimizing back-and-forth.*
