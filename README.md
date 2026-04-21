# LazyCoding

A collection of Agent Skills to help you write better code with LLMs — covering professional code generation and strict AI-driven code review.

## Skills

### Code Review (`skills/code-review/`)

AI-powered code review and bug fixing using the `lazyllm review` CLI.

| Skill | Description |
|-------|-------------|
| [SKILL.md](skills/code-review/SKILL.md) | Entry point — installation, model config, and all scenarios |
| [local-review.md](skills/code-review/local-review.md) | Review your own local changes before pushing (`lazyllm review-local`) |
| [pr-review.md](skills/code-review/pr-review.md) | Run multi-round AI review on a remote PR and post comments back to GitHub/GitLab/Gitee |
| [fix_bug.md](skills/code-review/fix_bug.md) | Fix bugs from review results — supports both local `review.json` and online PR comments |

## Philosophy

- **Write better code**: guide the AI to produce professional, clean, and minimal implementations
- **Review strictly**: catch real issues with multi-round LLM review, not just style linting
- **Fix systematically**: triage every review comment, apply reliable fixes, reject false positives with reasoning
