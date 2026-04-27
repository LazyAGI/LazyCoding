---
name: lazyllm-review
description: >-
  Run AI code review using the LazyLLM review CLI tool (lazyllm review /
  lazyllm review-local). Use when the user mentions: review, code review,
  review-local, review PR, fix review issues, post review to GitHub, lazyllm
  review, PR review, review 代码, 代码 review, review 问题, 修复 review,
  fix review comments, fix bug from review, 修复 review 评论, 处理 PR review 意见,
  根据 review 修 bug, 回复 review 评论。
  Covers installation, model/API-key config, local review JSON processing,
  GitHub auth, posting review comments to PRs, and fixing bugs from PR review comments.
---

# LazyLLM Review Skill

## Installation (minimal)

Core package plus MCP (required for DeepWiki and other MCP-based tools):

```bash
pip install lazyllm "mcp>=1.7.0"
```

Or use the bundled extra:

```bash
pip install "lazyllm[agent-advanced]"
```

For GitHub App authentication (JWT):

```bash
pip install PyJWT cryptography
```

> **Note:** After installing, check the version:
> ```bash
> pip show lazyllm | grep Version
> ```
> If the version is **≤ 0.7.6**, the `lazyllm review` and `lazyllm review-local` commands are not available yet.
> In that case, clone the latest code directly from GitHub and add the project root to `PYTHONPATH`:
>
> ```bash
> git clone https://github.com/LazyAGI/LazyLLM.git
> export PYTHONPATH=/path/to/LazyLLM:$PYTHONPATH
> ```
>
> Replace `/path/to/LazyLLM` with the actual clone path (e.g. `~/LazyLLM`).
> No installation needed — the `lazyllm review` / `lazyllm review-local` commands will work immediately.

---

## Model configuration

### API Key lookup order

The review tool calls `lazyllm.OnlineChatModule(source=..., model=...)`. Keys are resolved in this order:

1. `--base-url` flag paired with an environment variable (see below)
2. Environment variable `LAZYLLM_<SOURCE>_API_KEY` (e.g. `LAZYLLM_CLAUDE_API_KEY`, `LAZYLLM_OPENAI_API_KEY`, `LAZYLLM_QWEN_API_KEY`)
3. lazyllm config file field `<source>_api_key`

**Recommended: set an environment variable**

```bash
# Claude (default source)
export LAZYLLM_CLAUDE_API_KEY=sk-ant-xxx

# OpenAI
export LAZYLLM_OPENAI_API_KEY=sk-xxx

# Qwen
export LAZYLLM_QWEN_API_KEY=sk-xxx

# SiliconFlow / PPIO or any OpenAI-compatible proxy
export LAZYLLM_OPENAI_API_KEY=your-key
```

### Specifying model and base-url

```bash
# Claude (default)
lazyllm review-local --source claude --model claude-opus-4-6

# OpenAI
lazyllm review-local --source openai --model gpt-4o

# Custom proxy / base-url
lazyllm review-local --source openai --model gpt-4o \
  --base-url https://your-proxy.com/v1/
```

---

## Scenario 1: Review your own code (review-local)

See [local-review.md](local-review.md) for the full workflow.

### Quick commands

```bash
# Diff against main, including uncommitted changes (default)
lazyllm review-local --base main --output review.json

# Committed changes only (exclude working-tree changes)
lazyllm review-local --base main --no-uncommitted --output review.json

# Specify repo path
lazyllm review-local --repo-path /path/to/repo --base develop --output review.json
```

### How the AI tool should process review.json

1. Read `review.json`. Structure:
   - `instructions`: embedded AI processing guide — follow it directly
   - `pr_info`: branch info and PR summary
   - `stats`: issue counts (`total` / `critical` / `normal` / `suggestion`)
   - `issues[]`: each issue has `id`, `file`, `line`, `severity`, `category`, `problem`, `suggestion`

2. Process by severity: `critical` → `normal` → `suggestion`

3. For each issue:
   - Judge **Reliable** (valid concern) vs **Not Reliable** (misunderstanding / already fixed / invalid constraint)
   - Reliable → apply a fix at the indicated `file` + `line`
   - Not Reliable → record rejection reason, skip

4. Generate a final report using the template below

### Final report template

```markdown
# Review Fix Report

## Summary
- Total issues: N, Fixed: X, Rejected: Y

## Fixed issues
| ID | File | Line | Severity | Fix description |
|----|------|------|----------|-----------------|
| 1  | foo.py | 42 | critical | Added None guard to prevent null dereference |

## Rejected issues (Not Reliable)
| ID | Reason |
|----|--------|
| 3  | Already fixed in the previous commit |

## Unresolved issues
(If any, explain why)
```

---

## Scenario 2: Review someone else's PR (review)

See [pr-review.md](pr-review.md) for the full workflow.

### GitHub authentication

The tool resolves a token in this order:

1. Environment variable `GITHUB_TOKEN` or `GH_TOKEN`
2. `gh` CLI already logged in (`gh auth token`)
3. GitHub App (`app_id` + `installation_id` + PEM private key)

**Recommended: gh CLI**

```bash
gh auth login          # one-time browser auth
lazyllm review --pr 42 --repo owner/repo
```

**Or set a token directly**

```bash
export GITHUB_TOKEN=ghp_xxx
lazyllm review --pr 42 --repo owner/repo
```

### Post review comments to GitHub

Add `--post` to publish comments back to the PR:

```bash
lazyllm review --pr 42 --repo owner/repo --post
```

Without `--post`, results are printed locally only.

### Common flags

| Flag | Description | Default |
|------|-------------|---------|
| `--pr` | PR number (required) | — |
| `--repo` | `owner/name` format | `LazyAGI/LazyLLM` |
| `--backend` | `github`/`gitlab`/`gitee`/`gitcode` (auto-detect) | auto |
| `--source` | LLM source | `claude` |
| `--model` | Model name | `claude-opus-4-6` |
| `--base-url` | Proxy URL | official endpoint |
| `--language` | Output language `cn`/`en` | `cn` |
| `--post` | Post comments to PR | off |
| `--resume-from` | Resume from a specific stage | — |
| `--clear-checkpoint` | Clear checkpoint and restart | off |
| `--refresh-diff` | Fetch latest diff | off |

### Resuming from a checkpoint

```bash
# Restart from r1 (keeps clone/arch/spec cache)
lazyllm review --pr 42 --repo owner/repo --resume-from r1

# Full restart
lazyllm review --pr 42 --repo owner/repo --clear-checkpoint
```

Stage order: `clone → arch → spec → pr_summary → r1 → r2 → r3 → final → upload`

---

## Scenario 3: Fix bugs from PR review comments (fix-bug)

See [fix_bug.md](fix_bug.md) for the full workflow.

Use this scenario when the user says: "帮我修复 review 评论", "fix review comments", "处理 PR 上的 review 意见", "根据 review 修 bug", "fix bug from review", "回复 review 评论"。

### Quick steps

```bash
# 1. Fetch all review comments (line-level)
gh api repos/{OWNER}/{REPO}/pulls/{PR}/comments

# 2. Fetch overall reviews
gh api repos/{OWNER}/{REPO}/pulls/{PR}/reviews

# 3. Reply to a comment after fixing
gh api --method POST \
  repos/{OWNER}/{REPO}/pulls/{PR}/comments \
  --field in_reply_to={comment_id} \
  --field body="已修复：{简要说明}"
```

### AI processing rules

1. **Must use `gh` CLI** to fetch comments — never browse the PR page directly (collapsed comments will be missed)
2. For every comment, judge **Reliable** (fix it) vs **Not Reliable** (reject with reason)
3. Apply code fixes at the indicated `file` + `line`
4. Reply to every comment (fixed or rejected) via `gh api`
5. Generate a final fix report grouped by category (correctness / performance / architecture / style / maintainability)

---

## Other platforms

```bash
# GitLab
export GITLAB_TOKEN=glpat-xxx
lazyllm review --pr 10 --repo owner/repo --backend gitlab

# Gitee
export GITEE_TOKEN=xxx
lazyllm review --pr 5 --repo owner/repo --backend gitee
```

## Reference docs

- Local review full workflow: [local-review.md](local-review.md)
- PR review full workflow: [pr-review.md](pr-review.md)
- Fix bugs from PR review comments: [fix_bug.md](fix_bug.md)
