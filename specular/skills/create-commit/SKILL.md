---
name: create-commit
description: Stage and commit all current changes with a short conventional commit message. Invoked by the implement loop after a sub-issue is complete, not by users directly.
user-invocable: false
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *) Bash(git diff *)
---

# Commit

Stage and commit all changes with a conventional commit message.

## Message Format

```
<prefix>: <short lowercase description>
```

- **Prefix**: `feat:`, `fix:`, or `chore:` (pick the most appropriate)
  - `feat` - new functionality or capability
  - `fix` - bug fix or correction
  - `chore` - refactor, cleanup, deps, config, docs, tests, etc.
- **Description**: ultra-short, all lowercase, simple common english words

## Examples

```
feat: add user login page
fix: handle empty list in search
chore: update deps
chore: remove unused imports
feat: support dark mode
fix: prevent crash on missing config
chore: rename utils to helpers
```

## Workflow

1. Run `git diff HEAD` and `git status` to understand what changed
2. Pick the prefix based on the nature of the changes
3. Write the shortest accurate description possible
4. Run:

```bash
git add -A && git commit -m "<message>"
```
