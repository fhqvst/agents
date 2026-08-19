---
name: create-pr
description: Create a GitHub pull request using gh CLI with a short summary. Invoked by the implement loop in its terminal step, not by users directly.
user-invocable: false
allowed-tools: Bash(git push *) Bash(git log *) Bash(git diff *) Bash(gh pr create *) Bash(gh pr view *) Bash(gh repo view *)
---

# Open PR

## Instructions

1. Determine the base branch. Use the one you were given if the caller passed one. Otherwise detect it - don't assume `main`:

```bash
gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
```

2. Run `git log --oneline <base>..HEAD` and `git diff <base>...HEAD --stat` to understand what changed.
3. Write a PR body using this exact structure:

```
### Summary

In this PR we <one short sentence describing what was done>
```

- 1-2 sentences max for the summary
- Use plain, everyday words
- Don't explain why we made these changes or how they improve the codebase

For non-trivial PRs (multiple files changed, new logic, refactors), add a Context section after the summary. Skip it for small/obvious PRs (typos, renames, one-line fixes). The Context section gives reviewers additional context that isn't obvious from the diff alone - things that deserve scrutiny, security-sensitive logic, non-obvious decisions, or how to verify behavior. Don't describe what each file does (the diff already shows that). No bold labels or prefixes - just natural sentences, 1-3 bullet points.

Example of a good PR body:

```
### Summary

In this PR we add rate limiting to the public API endpoints

### Context

- The sliding window algorithm in `rateLimiter.ts` is the core logic - verify the edge case when the window resets mid-burst
- Try hitting `/api/prices` rapidly to confirm 429 responses kick in
```

4. Derive a short lowercase PR title from the commit (e.g. "feat: add barrel exports for components").
5. Push and create the PR:

```bash
git push -u origin HEAD
gh pr create --base <base> --title "<title>" --body "<body>" --assignee @me --draft
```

6. Return the PR URL.
