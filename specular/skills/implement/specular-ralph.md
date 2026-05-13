# Implement step

One iteration against parent issue `$PARENT`. Driver re-invokes you until you emit the stop sentinel.

# Pick a sub-issue

Fetch `$PARENT` via Linear MCP. The body has two parts: a human-facing RFC at the top (Problem, Proposal, Constraints, headline Implementation pseudocode) and a `+++ PLAN.md ... +++` collapsible at the bottom (user stories, vocabulary, deeper implementation decisions, testing decisions). **Both halves are the brief** - the top carries Problem/Proposal/Out-of-scope, the collapsible adds the rest. Parse the collapsible out of the body (content lives between the opener line and the next standalone `+++`) so you can pass the two pieces separately to `/specular:work-on-issue`.

List its sub-issues. Eligible = no `**Type:** HITL` line in the body AND every `blockedBy` is Done AND state is unstarted/started. Missing marker counts as AFK. Among eligible, prefer fewest open blockers, then earliest creation.

If the parent issue has zero sub-issues: tell user to run `/specular:plan`, emit sentinel, exit (no push, no PR).

If nothing eligible: jump to **Terminal step**.

# Worktree

Branch = parent's Linear `branchName`. Directory = branch with `/` → `-`, sibling of the current working tree (e.g. branch `fhqvst/pfg-869-foo` → directory `../fhqvst-pfg-869-foo`). Never use `/` in worktree directory names. Create with `git worktree add ../<dir> -b <branch>` if missing. `cd` in. Make sure dependencies are installed (whatever the project uses - `bun install`, `npm install`, `cargo fetch`, etc.).

# Bash hygiene (critical for headless permission matching)

This loop runs headless. The user's permission allowlist matches commands by literal prefix (`Bash(git checkout *)` matches `git checkout main` but NOT `git -C /path checkout main` or `cd /path && git checkout main`). To keep the loop unblocked:

- Never use `git -C <path> ...`. Always `cd` to the directory first, then run bare `git ...`.
- Never chain `cd ... && <cmd>` as a single Bash call. Run `cd <dir>` as its own Bash invocation, then run subsequent commands bare. The shell CWD persists across Bash tool calls within this session.
- Never combine commands with `&&`, `;`, or `|` if either side could otherwise match the allowlist on its own. One command per Bash call.
- Use relative paths once you've `cd`'d in. Bare `git status`, `git add .`, `git commit ...`, `bun install` - not absolute-path-prefixed variants.

# Work the sub-issue

Invoke `/specular:work-on-issue` with the sub-issue body, identifier, and the parent's full body as background (both the human-top and the parsed `PLAN.md` content). The agent will pick up vocabulary from `SPECULAR.md` files in the working tree itself (hierarchical, like `CLAUDE.md`).

# Validate (hard gate)

The repo's agent-instruction file (`CLAUDE.md` / `AGENTS.md` / `GEMINI.md`) names the lint, typecheck, and test commands. Run each one that applies. If any fails:

- Do NOT commit.
- Leave a comment on the sub-issue summarising the failure (command, exit code, last ~30 lines of output).
- End the turn without emitting the sentinel. Do not push.

If the agent-instruction file doesn't mention validation commands, leave a comment on the sub-issue asking the user to run `/specular:setup` to add them, and exit without committing.

Only proceed to commit when all configured validation commands pass cleanly.

# Commit, push, transition

Create the commit - message must reference the sub-issue identifier. If a user-defined `commit` skill is available, prefer it; otherwise fall back to `/specular:create-commit`. Then `git push -u origin <branch>`.

After a successful push, explicitly transition the sub-issue to **Done** via `mcp__plugin_linear_linear__save_issue` (look up the team's Done state via `mcp__plugin_linear_linear__list_issue_statuses` if you don't already know its id). We do this manually rather than relying on Linear's GitHub automation because the automation isn't always wired up.

End your turn. **Do NOT emit the sentinel** - more sub-issues may remain.

# Terminal step

Only when no eligible sub-issue. From the worktree, run `gh pr view --json number 2>/dev/null`. If it finds nothing, open the PR targeting `main` - prefer a user-defined `open-pr` skill if available, otherwise fall back to `/specular:create-pr`. If a PR already exists, skip creation. Then output exactly:

```
<promise>NO MORE TASKS</promise>
```

# Rules

- One sub-issue per iteration. Never batch.
- Never modify the parent (body, state, labels).
- Validation commands from the agent-instruction file are a hard gate - never commit on a red build.
- Never PR mid-loop - PR creation is terminal-only.
- On unexpected breakage (conflicts, missing branchName, broken main): leave a comment on the sub-issue, exit. No force-push, no destroying work.
