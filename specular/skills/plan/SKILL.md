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
- A `+++ PLAN.md ... +++` collapsible at the bottom adding the agent-only detail: user stories, deeper implementation decisions, testing decisions, further notes.

**The whole body is the source of truth.** Read both halves - the human-top carries the Problem, Proposal, Constraints, and Out-of-scope; the collapsible adds the rest. Parse the collapsible out of the body (content lives between the opener line `+++ PLAN.md` and the next standalone `+++` line) so you can reason about it separately when you need to.

If the `+++ PLAN.md` block is missing, warn the user that the parent wasn't created with `/specular:specify` and the breakdown will be coarser; work from the human-top alone.

### 3. Explore the codebase

Mandatory, scoped to the files and packages the slices will touch. Find the existing files, components, helpers, test files, and patterns each slice should build on, and note them by path - they go into the sub-issue bodies in step 6. Sub-issue titles and descriptions use the project's domain vocabulary as it appears in the code and the parent issue body.

You do this once here. Every implement, review, and fix agent would otherwise rediscover it from scratch, one per slice.

### 4. Draft vertical slices

Break the implementation plan into **tracer bullet** sub-issues. Each is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

Slices may be **HITL** (requires human interaction - architectural decision, design review) or **AFK** (can be implemented and merged unattended). Prefer AFK where possible.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Prefer many thin slices over few thick ones
- Prefer independent slices over a strict `blockedBy` chain where the seam allows it - the loop halts on the first failure, and a chain turns one failure into a stalled run
</vertical-slice-rules>

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

For each approved slice, create a Linear issue with `mcp__plugin_linear_linear__save_issue`. Set `parentId` to the parent issue identifier so it becomes a sub-issue. Set `state` to `Todo` (the slices have already been triaged through this skill). Inherit team, project, and assignee from the parent issue; if any are missing on the parent, fall back to whatever `SPECULAR.md`'s `## Linear` section specifies.

Mark HITL slices with a `**Type:** HITL` line in the body (see template below). AFK slices need no marker - the implement loop treats missing marker as AFK. Do not apply Linear labels for this; the body is the single source of truth.

Publish in dependency order (blockers first) so you can pass real identifiers to the `blockedBy` field for later slices.

**Each sub-issue is the complete brief for one agent.** The implement, review, and fix agents read the sub-issue and nothing else - they never fetch the parent. Anything the agent needs from the parent goes into the sub-issue: repeating parent text is fine. Each sub-issue is read by one agent once; the parent would otherwise be read by every agent on every slice.

Use this body template:

<issue-template>
**Type:** HITL

## Context

The parent's Problem in two sentences, then the Constraints "Out" items that apply to this slice, each as a bullet. Skip Out items that can't touch this slice.

## What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation. Quote verbatim any sentence, label, or template string from the parent that the slice must reproduce.

## Build on

- `path/to/existing/file.ts` - what it provides and how this slice uses it
- `path/to/Component.tsx` - the component or helper to reuse, not reinvent
- The pattern to follow, by path to an existing example

## Sources

For any copy or content that lives outside the repo: the exact URL and section heading. Quote it verbatim when short. Omit the section if nothing external is needed.

## Testing

- The parent's Testing Decisions that apply to this slice (e.g. "inline fixtures, not the real files")
- `path/to/existing.test.ts` - the test file to mirror

## Standards

- `standards/foo.md` - the specific standards docs to read for this slice, by path

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Blocked by

- A reference to the blocking sub-issue (if any)

Or "None - can start immediately" if no blockers.
</issue-template>

Omit the `**Type:**` line entirely on AFK slices. Every other section is required except Sources, which is omitted when empty.

Do NOT modify the parent issue's body or state.

Report the list of created sub-issue identifiers and URLs.
