# PR Review: Full Workflow

`lazyllm review` runs multi-round AI review on a remote PR (GitHub / GitLab / Gitee / GitCode) and optionally posts comments back to the PR.

---

## Authentication

### Option 1: gh CLI (GitHub — recommended)

```bash
# One-time login
gh auth login

# Then just run — no token config needed
lazyllm review --pr 42 --repo owner/repo
```

The tool calls `gh auth token` automatically.

### Option 2: Environment variable

```bash
# GitHub
export GITHUB_TOKEN=ghp_xxx
# or
export GH_TOKEN=ghp_xxx

# GitLab
export GITLAB_TOKEN=glpat-xxx

# Gitee
export GITEE_TOKEN=xxx

# GitCode
export GITCODE_TOKEN=xxx
```

### Option 3: GitHub App (CI / automation)

For running as a GitHub App in CI:

```bash
# Default private key path: ~/.config/lazyllm/github-app-private-key.pem
# Or override via env var:
export GITHUB_APP_PRIVATE_KEY_PATH=/path/to/private-key.pem
```

Programmatic usage:

```python
import lazyllm
git = lazyllm.Git(
    backend='github',
    app_id='123456',
    installation_id=789,
    private_key_path='/path/to/key.pem',
    repo='owner/repo',
)
```

---

## All flags

```
lazyllm review
  --pr NUMBER           PR number (required)
  --repo OWNER/NAME     Repository (default: LazyAGI/LazyLLM)
  --backend BACKEND     github/gitlab/gitee/gitcode (auto-detect)
  --source SOURCE       LLM source (default: claude)
  --model MODEL         Model name (default: claude-opus-4-6)
  --base-url URL        Proxy / custom API base URL
  --language cn|en      Review output language (default: cn)
  --post                Post comments to the PR (default: off)
  --keep-clone          Keep the cloned repo directory after review
  --clear-checkpoint    Clear checkpoint and restart
  --refresh-diff        Fetch latest PR head SHA and diff
  --resume-from STAGE   Resume from a specific stage
  --max-history-prs N   Max historical PRs to analyze for review spec (default: 20)
  --arch-cache PATH     Path to architecture doc cache JSON
  --spec-cache PATH     Path to review spec cache JSON
```

---

## Typical usage

### Local preview — no posting

```bash
lazyllm review --pr 42 --repo myorg/myrepo
```

Prints the review summary and issue count to the terminal; no comments are posted.

### Post comments to the PR

```bash
lazyllm review --pr 42 --repo myorg/myrepo --post
```

### Use a proxy or alternative model

```bash
export LAZYLLM_OPENAI_API_KEY=your-key
lazyllm review --pr 42 --repo myorg/myrepo \
  --source openai --model gpt-4o \
  --base-url https://your-proxy.com/v1/
```

### English output

```bash
lazyllm review --pr 42 --repo myorg/myrepo --language en
```

---

## Review pipeline stages

| Stage | Description |
|-------|-------------|
| `clone` | Clone repo, fetch diff |
| `arch` | Analyze overall repo architecture |
| `spec` | Extract review standards from historical PRs |
| `pr_summary` | Generate a summary of this PR |
| `r1` | Round 1: hunk-by-hunk initial review |
| `r2` | Round 2: design doc + deep analysis |
| `r3` | Round 3: cross-file correlation analysis |
| `final` | Aggregate, deduplicate, produce final comments |
| `upload` | Post comments to PR (`--post` only) |

Checkpoints are saved at `~/.lazyllm/review/cache/<repo>/<pr>/checkpoint.json`.

### Resuming from a checkpoint

```bash
# Restart from r1 (keeps clone/arch/spec cache)
lazyllm review --pr 42 --repo owner/repo --resume-from r1

# PR has new commits — refresh diff
lazyllm review --pr 42 --repo owner/repo --refresh-diff

# Full restart
lazyllm review --pr 42 --repo owner/repo --clear-checkpoint
```

---

## Reusing arch and spec cache across PRs

When reviewing multiple PRs in the same repo, reuse the architecture and spec cache to skip those stages:

```bash
# Cache is saved automatically on first run.
# For subsequent PRs, point to the cached files:
lazyllm review --pr 43 --repo owner/repo \
  --arch-cache ~/.lazyllm/review/cache/owner_repo/arch.json \
  --spec-cache ~/.lazyllm/review/cache/owner_repo/spec.json
```

---

## Output

Terminal output:

```
Reviewing PR #42 on owner/repo (model=claude-opus-4-6, language=cn)...
--- Review result ---
<PR summary>
Total issues: 8, posted: 8
```

- `Total issues`: number of issues found
- `posted`: comments successfully posted to the PR (`--post` only)

---

## Troubleshooting

**401 Unauthorized**

```
GitHub API returned 401 Unauthorized.
Please authenticate by running: gh auth login
```

→ Run `gh auth login` or set `GITHUB_TOKEN`.

**No token found**

```
No token for backend 'github'. Set token=... or env ('GITHUB_TOKEN', 'GH_TOKEN')
```

→ Set the corresponding environment variable or use `gh auth login`.

**Review interrupted**

Just re-run the same command — the checkpoint will resume from where it left off.
