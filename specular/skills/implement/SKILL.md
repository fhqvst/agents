---
name: implement
description: Run the autonomous implement loop on a Linear parent issue - iterates through its sub-issues, working on each in turn until all are done. Use when the user asks to "implement ABC-123" or "run implement".
argument-hint: "<parent-issue-id>"
---

# Implement loop

You are the **orchestrator** for parent issue `$ARGUMENTS`. You never write code, run tests, or edit files yourself - every change happens inside a subagent. You own exactly four things: the worktree, the order of work, pushing, and Linear state.

Keep your own context lean, but not blind. Read the sub-issue bodies - they're short, you need them to spot HITL markers, and they let you report on what actually landed.

Two things you never pull into your context: the parent's `+++ PLAN.md +++` collapsible, and diffs. Both are large and unbounded, and the subagents fetch what they need themselves.

## 1. Load the parent

Fetch the parent via Linear MCP. You need its `branchName` and its sub-issue list.

Zero sub-issues → tell the user to run `/specular:plan` first, and stop. No worktree, no PR.

## 2. Create the worktree

Resolve the base branch first - never assume `main`:

```
gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
```

Then `git fetch origin`. Both the worktree and the PR branch off this; hold onto the name for step 5.

Branch = the parent's Linear `branchName`. Directory = that branch with `/` replaced by `-`, as a sibling of the current working tree (branch `fhqvst/pfg-869-foo` → `../fhqvst-pfg-869-foo`). Never put `/` in the directory name.

If the directory doesn't exist: `git worktree add ../<dir> -b <branch> origin/<base>`. Branch explicitly off `origin/<base>`, not off whatever HEAD the user happened to be sitting on. If the branch already exists, drop the `-b` and the base: `git worktree add ../<dir> <branch>`.

Then `cd` into it and install dependencies once (`bun install`, `npm install`, `cargo fetch` - whatever the repo uses). Doing it here means the subagents don't each pay for it.

Record the absolute path. Every subagent needs it.

## 3. Order the work

Eligible sub-issue = no `**Type:** HITL` line in the body, every `blockedBy` is Done, and state is unstarted or started. A missing marker means AFK.

Order: fewest open blockers first, then earliest created. Compute this list once - the loop halts on failure, so the blocker graph can't shift under you mid-run.

Nothing eligible → skip to step 5.

## 4. For each sub-issue, in order

Spawn subagents with `subagent_type: "general-purpose"`. Never pass `isolation: "worktree"` - the worktree is already yours, and the slices share it deliberately.

### 4.1 Implement

Spawn with exactly this prompt:

```
Read and follow the instructions in ${CLAUDE_PLUGIN_ROOT}/skills/implement/implement.md.

Parent: <PARENT-IDENT>
Sub-issue: <SUB-ISSUE-IDENT>
Worktree: <ABSOLUTE-PATH>
```

Returns `DONE <IDENT> <sha>` or `FAILED <IDENT> <reason>`.

`FAILED` → **halt** (step 6). Don't move to the next sub-issue: later slices build on this one.

### 4.2 Review

Same shape, pointing at `${CLAUDE_PLUGIN_ROOT}/skills/implement/review.md`, with `Commit: <sha>` added.

Returns `NONE` or up to three findings.

### 4.3 Fix

Only when the reviewer returned findings. Check them against the sub-issue body first: drop any that reach beyond what the ticket asked for, and skip this step entirely if nothing survives. The reviewer is scoped to the ticket, but you're the one holding it. Point at `${CLAUDE_PLUGIN_ROOT}/skills/implement/fix.md`, pass the same parameters plus the findings verbatim.

Returns `FIXED <IDENT> <new-sha> <summary>` or `FAILED <IDENT> <reason>`. `FAILED` → **halt**.

### 4.4 Push and close

Only now, once the commit has settled:

1. `cd <worktree>` as its own Bash call, then `git push -u origin <branch>`.
2. Transition the sub-issue to **Done** via `mcp__plugin_linear_linear__save_issue`. Look the team's Done state id up once with `mcp__plugin_linear_linear__list_issue_statuses` and reuse it for the rest of the run. We do this by hand because Linear's GitHub automation isn't always wired up.

## 5. Open the draft PR

From the worktree, `gh pr view --json number,state`. Skip creation only if a PR exists **and** its state is `OPEN`. If there's none, or the most recent one is `CLOSED`/`MERGED`, open a new one against the base branch you resolved in step 2 - prefer a user-defined `open-pr` skill if one exists, otherwise `/specular:create-pr`, passing the base explicitly. It opens as a draft.

## 6. Report

Either way, tell the user: which sub-issues landed and what each one did, which failed and why, which HITL ones are still waiting, and the PR URL. Leave the worktree in place.

## Rules

- One sub-issue per subagent. Never batch.
- Subagents commit; only you push. That separation is what makes it safe for the fixer to amend.
- Never modify the parent issue (body, state, labels).
- Halt on the first failure. Never force-push, reset, or delete work to get unstuck.
- HITL sub-issues are never touched.
