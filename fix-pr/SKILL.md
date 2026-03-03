---
name: fix-pr
description: >
  Iteratively fixes a PR until Macroscope's GitHub check passes with zero
  unresolved review comments. Fetches findings, fixes code, pushes, and
  repeats. Use when the user wants to resolve all Macroscope review feedback
  on a PR.
license: MIT
compatibility: Requires git, gh (GitHub CLI) authenticated, and Macroscope installed on the repo.
metadata:
  author: byzkhan
  version: "1.0"
allowed-tools: Bash(gh:*) Bash(git:*)
---

# Fix PR

Resolve every Macroscope finding on a pull request automatically. The skill runs in a fix-verify loop — read the findings, patch the code, push, and confirm the check goes green — until nothing remains.

## Inputs

- **PR number** (optional): If not provided, detect the PR for the current branch.

## How it works

### 1. Discover the PR

```bash
gh pr view --json number,headRefName -q '{number: .number, branch: .headRefName}'
```

Grab owner/repo for GraphQL calls:
```bash
gh repo view --json owner,name -q '{owner: .owner.login, repo: .name}'
```

Switch to the PR branch if not already on it.

### 2. Fix-verify loop

Run up to **5 rounds**. Each round follows this sequence:

#### 2a. Push and wait for Macroscope

Push any pending changes, then poll until the Macroscope check finishes:

```bash
git push
gh pr view --json statusCheckRollup --jq '.statusCheckRollup[] | select(.name | test("Macroscope")) | "\(.status) \(.conclusion)"'
```

Hold until status is `COMPLETED`.

#### 2b. Collect open threads

Pull unresolved review threads via GraphQL (see [reference queries](references/graphql-queries.md)). Only look at threads authored by `macroscopeapp[bot]`.

#### 2c. Decide whether to stop

Exit the loop when:
- The Macroscope check is green (SUCCESS / NEUTRAL / SKIPPING) **and** there are no unresolved threads, **or**
- 5 rounds have been exhausted.

#### 2d. Patch the code

Walk through each unresolved finding:

1. Open the file and read the surrounding context.
2. If the comment refers to a line that has already been rewritten (stale diff position), confirm the underlying issue is gone and queue the thread for resolution.
3. Otherwise, apply the fix. Prefer targeted edits over full-file rewrites.
4. For false positives that genuinely cannot be addressed, still resolve the thread so it doesn't block the next round.

Keep related changes in the same commit — avoid micro-commits for individual line fixes.

#### 2e. Clean up resolved threads

Any thread whose underlying issue is already fixed in the working tree but still shows as open on GitHub gets resolved with a GraphQL mutation:

```bash
gh api graphql -f query='
mutation {
  t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
}'
```

Batch them into a single mutation when there are several.

#### 2f. Commit, push, next round

```bash
git add <changed-files>
git commit -m "Address Macroscope findings (round N)"
git push
```

Return to step 2a.

### 3. Summary

Print a short status block when finished:

```
fix-pr done  ·  2 rounds  ·  7 resolved  ·  0 remaining  ·  check: pass
```

If the loop hit the cap, list whatever is still open:

```
fix-pr stopped after 5 rounds  ·  12 resolved  ·  2 remaining  ·  check: pass

Open findings:
  src/lib.rs:45 — "Consider handling the error case"
  src/main.js:112 — "Potential XSS via innerHTML"
```

## Guidelines

- Address every finding — don't skip any.
- Read a file before editing it.
- Group fixes into logical commits.
- Never force-push or rewrite history.
- Only resolve a thread via GraphQL when the code genuinely already addresses it.
- After 3 failed attempts on the same finding, surface it to the user instead of looping further.
