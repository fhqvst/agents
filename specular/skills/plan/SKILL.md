---
name: plan
description: Break a parent Linear issue (an RFC produced by /specular:specify) into independently-grabbable sub-issues using vertical slices. Use when the user wants to break an RFC into sub-issues, split a parent issue into vertical slices, create tracer-bullet tasks, or carve up implementation work.
argument-hint: "<parent-issue-id>"
---

> **PR model is fixed**: one PR per parent issue, one commit per sub-issue, all sub-issues land on the same branch. Do not ask the user how to structure PRs.

# Plan

Create Linear **sub-issues** under a parent issue using vertical slices (tracer bullets).

## Process

### 1. Identify the parent issue

The parent issue identifier (e.g. `ABC-123`) is passed as `$ARGUMENTS`. If empty, fall back to one already established earlier in the conversation (typically because `/specular:specify` just created it). If neither is available, ask the user before proceeding.

### 2. Load the parent context

Fetch the parent issue with `mcp__plugin_linear_linear__get_issue`. The body has two parts produced by `/specular:specify`:

- A terse **human-facing RFC** at the top: Problem, optional Background, Proposal, Constraints (In/Out), Implementation (headline pseudocode), References.
- A `+++ PLAN.md ... +++` collapsible at the bottom adding the agent-only detail: user stories, vocabulary, deeper implementation decisions, testing decisions, further notes.

**The whole body is the source of truth.** Read both halves - the human-top carries the Problem, Proposal, Constraints, and Out-of-scope; the collapsible adds the rest. Parse the collapsible out of the body (content lives between the opener line `+++ PLAN.md` and the next standalone `+++` line) so you can reason about it separately when you need to.

If the `+++ PLAN.md` block is missing, warn the user that the parent wasn't created with `/specular:specify` and the breakdown will be coarser; work from the human-top alone.

### 3. Explore the codebase (optional)

If you have not already explored the relevant code, do so. Sub-issue titles and descriptions should use the project's domain vocabulary - assemble it from the relevant `SPECULAR.md` files (see [../specify/SPECULAR-FORMAT.md](../specify/SPECULAR-FORMAT.md)). If `PLAN.md` has a Vocabulary section, treat its terms as proposed canonical language for this change.

### 4. Draft vertical slices

Break the implementation plan into **tracer bullet** sub-issues. Each is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

Slices may be **HITL** (requires human interaction - architectural decision, design review) or **AFK** (can be implemented and merged unattended). Prefer AFK where possible.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Prefer many thin slices over few thick ones
</vertical-slice-rules>

#### Vocabulary slice (optional)

If `PLAN.md` has a Vocabulary section that introduces new terms or sharpens existing ones, also draft a dedicated AFK slice titled "Update SPECULAR.md vocabulary". Its job is to propagate those terms into the appropriate `SPECULAR.md` file(s) - creating new ones, editing existing ones, or splitting a parent file when a deeper folder needs to override.

Skip this slice if the vocabulary changes are trivial (one rename) or absent. When in doubt, include it - vocabulary debt compounds quickly.

Order this slice **first** in the dependency graph if other slices use the new terminology, so subsequent commits land with the canonical vocabulary already in place.

### 5. Quiz the user

Present the proposed breakdown as a numbered list. For each slice, show:

- **Title**: short descriptive name
- **Type**: HITL / AFK
- **Blocked by**: which other slices (if any) must complete first
- **User stories covered**: which user stories from `PLAN.md` this addresses

Ask:

- Does the granularity feel right? (too coarse / too fine)
- Are the dependencies correct?
- Should any slices be merged or split?
- Are HITL/AFK markers correct?

Iterate until the user approves.

### 6. Publish the sub-issues

For each approved slice, create a Linear issue with `mcp__plugin_linear_linear__save_issue`. Set `parentId` to the parent issue identifier so it becomes a sub-issue. Set `state` to `Todo` (the slices have already been triaged through this skill). Inherit team, project, and assignee from the parent issue; if any are missing on the parent, fall back to whatever the repo's agent-instruction file (`CLAUDE.md` / `AGENTS.md` / `GEMINI.md`) specifies.

Mark HITL slices with a `**Type:** HITL` line in the body (see template below). AFK slices need no marker - the implement loop treats missing marker as AFK. Do not apply Linear labels for this; the body is the single source of truth.

Publish in dependency order (blockers first) so you can pass real identifiers to the `blockedBy` field for later slices.

Use this body template:

<issue-template>
**Type:** HITL

## What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Blocked by

- A reference to the blocking sub-issue (if any)

Or "None - can start immediately" if no blockers.
</issue-template>

Omit the `**Type:**` line entirely on AFK slices.

Do NOT modify the parent issue's body or state.

Report the list of created sub-issue identifiers and URLs.
