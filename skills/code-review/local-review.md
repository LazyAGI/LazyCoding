# review-local: Full Workflow

`lazyllm review-local` runs AI review on a local git repository. It uses `git merge-base` to compute the diff of the current branch against a base branch, then writes results to a JSON file.

---

## All flags

```
lazyllm review-local
  --repo-path PATH      Path to local git repo (default: current directory)
  --base BRANCH         Base branch to diff against (default: main)
  --no-uncommitted      Exclude uncommitted changes (default: include)
  --source SOURCE       LLM source (default: claude)
  --model MODEL         Model name (default: claude-opus-4-6)
  --base-url URL        Proxy / custom API base URL
  --language cn|en      Review output language (default: cn)
  --output PATH         Output JSON path (default: review.json)
  --clear-checkpoint    Clear checkpoint and restart from scratch
```

---

## Typical usage

### Pre-commit self-review

```bash
cd /path/to/my-project
lazyllm review-local --base main --output review.json
```

### Committed changes only (ignore working tree)

```bash
lazyllm review-local --base main --no-uncommitted --output review.json
```

### Diff against a non-main base branch

```bash
lazyllm review-local --base develop --output review.json
```

---

## review.json structure

```json
{
  "instructions": "...(AI processing guide — follow directly)",
  "pr_info": {
    "branch": "feature/xxx",
    "base": "main",
    "summary": "AI-generated summary of this change"
  },
  "stats": {
    "total": 5,
    "critical": 1,
    "normal": 3,
    "suggestion": 1
  },
  "issues": [
    {
      "id": 1,
      "file": "src/foo.py",
      "line": 42,
      "severity": "critical",
      "category": "correctness",
      "problem": "Description of the problem",
      "suggestion": "Suggested fix"
    }
  ]
}
```

**Severity levels:**
- `critical`: must fix (logic errors, security vulnerabilities, data-loss risk)
- `normal`: should fix (performance, maintainability)
- `suggestion`: optional improvement

**Categories:**
`correctness` / `performance` / `architecture` / `style` / `maintainability`

---

## Full AI processing workflow

### Step 1: Read and understand issues

Read `review.json`. Check `stats` for scale, then go through `issues` one by one.

### Step 2: Process by priority

```
critical → normal → suggestion
```

### Step 3: Judge each issue

| Situation | Judgment | Action |
|-----------|----------|--------|
| Real bug or security issue | Reliable | Fix the code |
| Already fixed elsewhere | Not Reliable | Record, skip |
| AI misunderstood the logic | Not Reliable | Record rejection reason |
| Pre-existing issue (not introduced by this change) | Case-by-case | Fix if trivial, open issue if complex |
| Style conflict with project conventions | Not Reliable | Record, skip |

### Step 4: Locate and fix

Use `file` + `line` to locate the code, apply the fix guided by `suggestion`:
- Only change code directly related to the issue
- Do not introduce unrelated logic changes
- Preserve the existing code style

### Step 5: Generate report

```markdown
# Review Fix Report

## Overview
- Issues scanned: N (critical: X, normal: Y, suggestion: Z)
- Fixed: A
- Rejected: B (Not Reliable)
- Skipped: C (suggestion level, optional)

## Fixed issues

### [critical] Issue #1 — src/foo.py:42
**Problem**: Null dereference — crashes when result is None
**Fix**: Added None guard, returns default value

### [normal] Issue #3 — src/bar.py:88
**Problem**: Connection created inside loop, poor performance
**Fix**: Moved connection creation outside the loop

## Rejected issues

### Issue #2 — Not Reliable
**Reason**: False positive — the variable is guaranteed non-null by the caller at caller.py:15

## Unresolved issues (suggestion level, optional)
- Issue #5: Consider adding type annotations (low priority, can be addressed later)
```

---

## Resuming from a checkpoint

`review-local` saves checkpoints automatically. If interrupted, just re-run the same command to resume.

Force a full restart:

```bash
lazyllm review-local --base main --clear-checkpoint --output review.json
```
