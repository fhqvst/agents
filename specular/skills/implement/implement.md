# Implement a sub-issue

You implement exactly one sub-issue, test-first, inside a worktree that already exists. The orchestrator gave you:

- `Parent:` the parent Linear issue identifier
- `Sub-issue:` the sub-issue identifier to implement
- `Worktree:` absolute path to the worktree

## 1. Enter the worktree

`cd <Worktree>` as its own Bash call, before anything else. Every later command runs bare from there. Work done outside the worktree is lost.

## 2. Read the brief

Fetch the parent via Linear MCP. Its body has two halves and **both are the brief**:

- the human-facing RFC at the top (Problem, Proposal, Constraints, headline Implementation pseudocode)
- a `+++ PLAN.md ... +++` collapsible at the bottom (user stories, deeper implementation decisions, testing decisions). Its content lives between the opener line and the next standalone `+++`.

Then fetch the sub-issue. That body is what you implement; the parent is background.

Read the repo's own standards (`CLAUDE.md`, `AGENTS.md`, and anything they point at) and follow them.

## 3. Implement

Invoke `/specular:work-on-issue` with the sub-issue body, its identifier, and both halves of the parent body as background.

## 4. Validate (hard gate)

Run the project's lint, typecheck, and test commands. Detect them from project files (`package.json` scripts, `Cargo.toml`, `Makefile`, `justfile`, `pyproject.toml`, etc.) - whatever the repo uses. Skip categories that don't apply.

If any command fails: do **not** commit. Leave a comment on the sub-issue with the command, exit code, and last ~30 lines of output. Then return `FAILED`.

Only commit when every applicable command passes cleanly.

## 5. Commit - do not push

Prefer a user-defined `commit` skill if one exists; otherwise `/specular:create-commit`. The message must reference the sub-issue identifier.

**Never push.** A reviewer runs after you and may hand your work to a fixer who amends this commit; the orchestrator pushes once that settles. **Never transition the issue in Linear** either - the orchestrator owns Linear state.

On unexpected breakage (conflicts, broken base branch, missing files): comment on the sub-issue and return `FAILED`. Never force-push, reset, or delete work to get unstuck.

## Bash hygiene

Permissions match commands by literal prefix, so:

- Never `git -C <path> ...`. You already `cd`'d in; run bare `git ...`.
- Never chain with `&&`, `;`, or `|`. One command per Bash call.
- Use relative paths inside the worktree.

## Return

Your final message must be exactly one line and nothing else:

```
DONE <SUB-ISSUE-IDENT> <commit-sha>
```

```
FAILED <SUB-ISSUE-IDENT> <one-line reason>
```

Detail belongs in the Linear comment, not in this line.
