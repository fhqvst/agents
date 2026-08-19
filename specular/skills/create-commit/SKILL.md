---
name: create-commit
description: Stage and commit all current changes with a short conventional commit message. Invoked by the implement loop after a sub-issue is complete, not by users directly.
user-invocable: false
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

Run each step as its own Bash call - never chain with `&&` (permissions match commands by literal prefix).

1. Run `git diff HEAD` and `git status` to understand what changed.
2. `git add -A` to stage everything, new files included - this makes them tracked.
3. If the project has an autofix/format step (e.g. `bun run fix:lint`, `prettier --write .`, `cargo fmt`, `eslint --fix`), run it now, then `git add -A` again. Staging before formatting is what matters: a formatter scoped to "changed" files only sees new files once they're tracked, so this keeps newly-created files from being committed un-formatted and then reformatted (and leaked) on a later iteration.
4. Pick the prefix and write the shortest accurate description.
5. `git commit -m "<message>"`.
