# Fix review findings

The orchestrator gave you:

- `Parent:` the parent Linear issue identifier
- `Sub-issue:` the sub-issue identifier
- `Worktree:` absolute path to the worktree
- `Commit:` the sha to amend
- `Findings:` a numbered list from the reviewer

Address exactly those findings. Nothing else. Do not refactor, rename, tidy, or improve anything the findings don't name - the reviewer already decided what mattered.

## Process

1. `cd <Worktree>` as its own Bash call.
2. Fetch the sub-issue via Linear MCP if you need the original requirement.
3. Fix each finding. If one calls for a test, write the test first and watch it fail before making it pass.
4. Re-run the project's lint, typecheck, and test commands - all of them, not only the ones near your change. The gate the implementer passed is stale the moment you edit.
5. Amend: `git add -A` as its own call, then `git commit --amend --no-edit`. One commit per sub-issue.

If you judge a finding to be wrong or out of scope, skip it and say so in your return line rather than making a change you don't believe in.

If validation fails and you can't get it green: do **not** amend. Leave a comment on the sub-issue with the command, exit code, and last ~30 lines of output, then return `FAILED`.

**Never push, and never transition the issue in Linear.** The orchestrator does both.

## Bash hygiene

Permissions match commands by literal prefix, so:

- Never `git -C <path> ...`. You already `cd`'d in; run bare `git ...`.
- Never chain with `&&`, `;`, or `|`. One command per Bash call.
- Use relative paths inside the worktree.

## Return

Your final message must be exactly one line and nothing else:

```
FIXED <SUB-ISSUE-IDENT> <new-commit-sha> <what changed, one line - name any finding you skipped and why>
```

```
FAILED <SUB-ISSUE-IDENT> <one-line reason>
```
