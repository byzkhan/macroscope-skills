# Macroscope Skills for Claude Code

Claude Code skills for [Macroscope](https://macroscope.com) code review.

## Available Skills

### fix-pr

Iteratively fixes a PR until Macroscope's GitHub check passes with zero unresolved review comments. Fetches findings, fixes code, pushes, and repeats.

**Install:**

```
/install-skill https://github.com/byzkhan/macroscope-skills/tree/main/fix-pr
```

**Usage:**

```
/fix-pr
```

Run this on any branch with an open PR that has Macroscope installed. It will:

1. Wait for the Macroscope check to complete
2. Fetch all unresolved review comments
3. Fix each finding in the code
4. Commit, push, and repeat until clean

**Requirements:**

- [Claude Code](https://claude.ai/claude-code) installed
- `gh` (GitHub CLI) authenticated
- [Macroscope](https://macroscope.com) installed on the repo

## License

MIT
