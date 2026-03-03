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

Iteratively fix a PR until Macroscope gives it a clean review: check passes, zero unresolved comments.

## Inputs

- **PR number** (optional): If not provided, detect the PR for the current branch.

## Instructions

### 1. Identify the PR

```bash
gh pr view --json number,headRefName -q '{number: .number, branch: .headRefName}'
```

Extract owner/repo:
```bash
gh repo view --json owner,name -q '{owner: .owner.login, repo: .name}'
```

Switch to the PR branch if not already on it.

### 2. Loop

Repeat the following cycle. **Max 5 iterations** to avoid runaway loops.

#### A. Wait for Macroscope check

Push the latest changes (if any) and wait for Macroscope's check to complete:

```bash
git push
```

Poll until the Macroscope check finishes:
```bash
gh pr view --json statusCheckRollup --jq '.statusCheckRollup[] | select(.name | test("Macroscope")) | "\(.status) \(.conclusion)"'
```

Wait if `IN_PROGRESS` or `QUEUED`. Proceed once `COMPLETED`.

#### B. Fetch unresolved review threads

Use GraphQL to get unresolved threads efficiently (see [GraphQL reference](references/graphql-queries.md)):

```bash
gh api graphql -f query='...' --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false)'
```

Filter to comments from `macroscopeapp[bot]`.

#### C. Check exit conditions

Stop the loop if **any** of these are true:
- Macroscope check passed (SUCCESS/NEUTRAL/SKIPPING) AND **zero unresolved comments**
- Max iterations reached (report current state)

#### D. Fix actionable comments

For each unresolved Macroscope comment:

1. Read the file and understand the comment in context.
2. Check if the comment points to a stale diff line (code already changed). If the fix is already applied, mark it for resolution in Step E.
3. If actionable, make the fix.
4. If it's a false positive, note it but still resolve the thread.

Group related fixes together — don't make one commit per line change.

#### E. Resolve stale threads

For threads where the code fix is already applied but the thread wasn't auto-resolved, resolve via GraphQL:

```bash
gh api graphql -f query='
mutation {
  t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
}'
```

Batch multiple resolutions into a single mutation when possible.

#### F. Commit and push

```bash
git add <specific-files>
git commit -m "Fix Macroscope review findings (iteration N)"
git push
```

Then go back to step **A**.

### 3. Report

After exiting the loop, summarize:

| Field | Value |
|-------|-------|
| Iterations | N |
| Check status | PASS/FAIL |
| Comments resolved | N |
| Remaining comments | N (if any) |

If the loop exited due to max iterations, list remaining unresolved comments and suggest next steps.

## Output format

```
Fix-PR complete.
  Iterations:    2
  Check:         PASS
  Resolved:      7 comments
  Remaining:     0
```

If not fully resolved:

```
Fix-PR stopped after 5 iterations.
  Check:         PASS
  Resolved:      12 comments
  Remaining:     2

Remaining issues:
  - src/lib.rs:45 — "Consider handling the error case"
  - src/main.js:112 — "Potential XSS via innerHTML"
```

## Rules

- Fix every finding — never skip one
- Always read a file before editing it
- Batch fixes into logical commits, not one per finding
- Never force-push or rewrite history
- Only resolve a thread via GraphQL if the code fix is genuinely already applied
- If a fix fails after 3 attempts, flag it to the user
