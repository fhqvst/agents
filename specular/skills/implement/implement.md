# Implement a sub-issue

You implement exactly one sub-issue, test-first, inside a worktree that already exists. The orchestrator gave you:

- `Sub-issue:` the sub-issue identifier to implement
- `Worktree:` absolute path to the worktree

## 1. Enter the worktree

`cd <Worktree>` as its own Bash call, before anything else. Every later command runs bare from there. Work done outside the worktree is lost.

## 2. Read the brief

Fetch the sub-issue via Linear MCP. Its body is the whole brief: Context, What to build, Build on, Sources, Testing, Standards, Acceptance criteria. **Do not fetch the parent issue.** It is large, and everything you need from it was written into the sub-issue.

If the sub-issue lacks a section you need (a source for copy, a testing decision, which file to build on), comment on the sub-issue naming what is missing and return `FAILED`. Do not read the parent to fill the gap; the fix belongs in the sub-issue.

The repo's root `CLAUDE.md` is already in your context - the harness loads it at spawn. Do not `cat` it again, and do not read `AGENTS.md` (it is usually the same file). Read only the standards docs the sub-issue's Standards section lists.

## 3. Implement

Invoke `/specular:work-on-issue` with the sub-issue body and its identifier.

## 4. Validate (hard gate)

Run the project's lint, typecheck, and test commands. Detect them from project files (`package.json` scripts, `Cargo.toml`, `Makefile`, `justfile`, `pyproject.toml`, etc.) - whatever the repo uses. Skip categories that don't apply.

If any command fails: do **not** commit. Leave a comment on the sub-issue with the command, exit code, and last ~30 lines of output. Then return `FAILED`.

Only commit when every applicable command passes cleanly.

## 5. Commit - do not push

Prefer a user-defined `commit` skill if one exists; otherwise `/specular:create-commit`. The message must reference the sub-issue identifier.

**Never push.** A reviewer runs after you and may hand your work to a fixer who amends this commit; the orchestrator pushes once that settles. **Never transition the issue in Linear** either - the orchestrator owns Linear state.

On unexpected breakage (conflicts, broken base branch, missing files): comment on the sub-issue and return `FAILED`. Never force-push, reset, or delete work to get unstuck.

## Budget

Every turn re-reads your whole context, so cost is turns times context. Keep both down:

- Ramp-up target: first edit within ~15 tool calls. Build on and Testing name the files to open; open those, not the tree.
- During red-green, run only the test file you are working in. The full suite runs once, at the gate.
- Read a file once. Do not re-`cat` something already in your context.
- Hard stop: if you are past roughly 120 tool calls and the gate is not green, stop. Comment on the sub-issue with where you got stuck and return `FAILED`. A slice that needs more than that is mis-sized, and grinding on burns far more than restarting.

## File edits

Use the `Edit` and `Write` tools for file changes, even when the harness tells you to prefer Bash. Output tokens are the wall clock, and a heredoc or a patch script that carries old and new strings emits the code twice. Never rewrite a whole file you already wrote this run; edit it.

## Bash hygiene

- Never `git -C <path> ...`. You already `cd`'d in; run bare `git ...`.
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
