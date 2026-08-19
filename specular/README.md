<br />

<h1 align="center"><code>/specular</code></h1>

<p align="center">
  A Claude Code plugin for agent-driven development, based on Matt Pocock's <a href="https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs"><code>/grill-with-docs</code></a> skill.
</p>

<br />

## Introduction

- **No clutter.** Specs live in Linear. The only footprint is a single `SPECULAR.md` (in your repo root, or a parent folder if you keep sibling worktrees) for Specular's own config.
- **Three commands.** `/specular:specify` → `/specular:plan` → `/specular:implement`. No config, no hidden state.
- **RFCs for humans.** Specs in two parts: an RFC that takes 1 minute to read + a verbose agent-facing brief in a collapsible section.
- **Specs over vibes.** `specify` grills you until the brief is sharp enough to act on - ambiguous nouns, undefined verbs, and hand-wavy edges surfaced up front.

#### Without Specular

```text
you:    "add notifications somewhere"
agent:  *builds the wrong thing in record time*
you:    "no, not like that"
agent:  *builds a different wrong thing*
```

#### With Specular

```text
/specular:specify "add notifications somewhere"
  → grilling: which event? which user? in-app or email? batched or instant?
  → Linear RFC published with sharp language and an agent-facing PLAN.md

/specular:plan ABC-123
  → vertical slices, each marked AFK (autonomous) or HITL (human-in-the-loop) in the issue body

/specular:implement ABC-123
  → per slice: subagent implements (TDD) → reviewer checks it → fixer amends → push
  → opens a draft PR once every AFK slice is done
```

## Contents

- [Getting started](#getting-started)
- [How it works](#how-it-works)
  - [RFC format](#rfc-format)
  - [`SPECULAR.md`](#specularmd)
- [API](#api)
- [FAQ](#faq)
- [References](#references)

## Getting started

> [!IMPORTANT]
> The implement loop does its work in subagents, and every un-approved tool call interrupts it for a permission prompt. `/specular:setup` figures out what to allow from your project's runner (`bun`, `cargo`, etc.) and writes the allowlist for you - run it before `/specular:implement` if you want it to run unattended.

1. **Authenticate the Linear MCP.** In Claude Code, run `/plugin`, install the Linear plugin, and sign in. Specular speaks to Linear through this MCP.

2. **Authenticate the GitHub CLI:**

   ```sh
   gh auth login
   ```

3. **Install the plugin** in Claude Code:

   ```sh
   /plugin marketplace add fhqvst/agents
   /plugin install specular@fhqvst
   ```

4. **Drive it:**

   ```sh
   /specular:setup                  # one-time per repo: writes SPECULAR.md config + Bash allowlist
   /specular:specify "..."          # produces a parent Linear issue (RFC) from a description, or pass an existing issue ID to sharpen it in place
   /specular:plan ABC-123           # breaks the parent into sub-issues
   /specular:implement ABC-123      # runs the loop until done, then opens a PR
   ```

> [!TIP]
> **Optional MCPs for UI work.** If you have the [Playwright MCP](https://github.com/microsoft/playwright-mcp) and/or the [Figma MCP](https://www.figma.com/blog/introducing-figmas-dev-mode-mcp-server/) installed, `/specular:specify` will use them automatically when the change touches UI. Playwright lets it screenshot the running app to ground the grilling session in the current UI; Figma lets it build a low-fi clickable wireframe (gray rectangles only) and attach the prototype URL to the Linear issue as a preview card. Both are inferred from the seed prompt - no flags needed. If the MCPs aren't installed, `/specify` proceeds normally without them.

## How it works

Specular is a three-step pipeline against Linear. You drive each step; the agent only writes what's already been approved on the previous one.

**1. Specify.** You hand `/specular:specify` a rough idea. It grills you - poking at fuzzy terminology, undefined behavior, hidden assumptions, and missing constraints. The result is a parent Linear issue (the "RFC") with two layers: a short human-readable RFC at the top for teammates, and a verbose agent-facing brief in a collapsible at the bottom for Specular itself.

**2. Plan.** `/specular:plan ABC-123` reads the RFC and slices it into vertical sub-issues. Each one is classified in its body as **AFK** (safe to implement unattended - the default, no marker needed) or **HITL** (needs you in the loop, marked with a `**Type:** HITL` line). No Linear labels involved - the body is the single source of truth.

**3. Implement.** `/specular:implement ABC-123` orchestrates a loop of subagents on your machine. It sets up one worktree for the parent, then walks the eligible AFK sub-issues in order. Each one gets a subagent that implements it test-first and runs your lint/typecheck/test as a hard gate, a second that reviews the commit against the ticket, and - only if that review found something worth acting on - a third that amends the commit. Then the orchestrator pushes and transitions the sub-issue to **Done** in Linear. Once every AFK sub-issue is done, it opens a draft PR. HITL sub-issues stay open for you to handle when you're back.

The complete footprint:

- **In your repo.** `SPECULAR.md` (in your repo root, or a parent folder if you keep sibling worktrees - see below) and `.claude/settings.local.json` are written only during `/specular:setup`, with a diff shown before each write. Specular never modifies `CLAUDE.md` / `AGENTS.md` / `GEMINI.md`. Source code, tests, and other implementation files are touched as part of each sub-issue's commit - same as any agent-driven change - on a dedicated worktree branch named after Linear's `branchName`.
- **On Linear.** Creates the parent issue and sub-issues, and transitions sub-issues to **Done** as the loop completes them. Does **not** modify workspace settings, workflow states, teams, projects, members, integrations, labels, or any other configuration. The plugin uses the Linear MCP as a regular user; it cannot escalate beyond what you can do in the Linear UI.
- **On GitHub.** One draft PR per parent issue, opened only after every AFK sub-issue is done. No comments, no labels, no other API calls.
- **On your machine.** A git worktree as a sibling of your CWD (`../<linear-branch-name>`) for the implement loop. You remove it when you're done with the branch.

> [!TIP]
> For the commit and PR steps, the loop prefers a user-defined `commit` or `open-pr` skill if one is available (e.g. in `~/.claude/skills/` or `.claude/skills/`), and falls back to Specular's bundled defaults otherwise. If you already have your own conventions wired up as skills, Specular will just use them.

### RFC format

The parent Linear issue body has a fixed shape, inspired by Basecamp's Shape Up but heavily compressed for skim-readability:

- **Problem** - one or two sentences. What hurts.
- **Background** - the minimum context a reader needs to evaluate the proposal.
- **Proposal** - what we're going to do, at the right altitude (not code).
- **Constraints** - explicit non-goals, scope cuts, "rabbit holes" we're refusing to go down.
- **Implementation** - high-level shape of the work, enough to plan from.
- **References** - links: prior issues, docs, dashboards, conversations.

Below that, a `PLAN.md` collapsible holds the full agent-facing brief - sharper invariants, deeper implementation decisions, testing decisions, anything verbose enough to drown out the human-readable RFC at the top. Teammates read the top; Specular reads the bottom.

### `SPECULAR.md`

A single `SPECULAR.md` holds Specular's config in prose. Skills find it by walking upward from your current working directory, stopping at the first hit (or `$HOME` / the filesystem root). That means it can live either in the repo root, or - if you use a parent-folder worktree layout like `~/Development/foo/{main,branch-1,...}` - in the parent folder, so every sibling worktree picks up the same config without duplication. `/specular:setup` writes the file to whatever directory you invoke it from.

Example contents:

```md
# Specular

## Linear

### Q2 Roadmap (`<project-id>`)

- Assignee: filip (`<user-id>`)

### Web (`<project-id>`)

- Paths: `apps/web/**`, `packages/web-ui/**`
- Assignee: alice (`<user-id>`)

### SDK (`<project-id>`)

- Paths: `packages/sdk/**`
```

`/specular:specify` and `/specular:plan` read the `## Linear` section to decide where to file issues. Each `###` subsection is one Linear project; `Paths` are the globs it owns. The project with no `Paths` is the implicit catch-all when nothing matches. The implement loop detects lint/typecheck/test commands per project (from `package.json`, `Cargo.toml`, etc.) and runs them as a hard gate before each commit - no need to declare them here.

It's a separate file (rather than a section in `CLAUDE.md` / `AGENTS.md`) on purpose - those files are shared real estate with whatever other agent tooling you use, and we don't want Specular to squat on them. If you swap Specular out later, deleting `SPECULAR.md` cleanly removes its footprint.

## API

### `/specular:setup` *(one-time per repo)*

Creates or updates `SPECULAR.md` (writing to the current working directory if it doesn't already exist somewhere up the tree) and confirms two things: which Linear **projects** new issues file into (plus optional **glob-based path routing** for monorepos), and that `.claude/settings.local.json` pre-approves the Bash patterns the loop will need (`git`, `gh`, plus the project's runner like `bun` / `cargo` / `pnpm`). Anything missing, setup offers to write in. Never modifies `CLAUDE.md` / `AGENTS.md` / `GEMINI.md`.

### `/specular:specify`

You hand it an idea - either a freeform prompt (`/specular:specify "..."`) or an existing Linear ticket ID (`/specular:specify ABC-123`) when you've already drafted the proposal in Linear and just want Specular to sharpen it. Specular then **grills you** before writing anything down - this is the most important step in the workflow. It probes for fuzzy terminology, undefined behavior, hidden assumptions, and missing constraints, and keeps asking until the spec is sharp.

Once the idea is sharp, Specular publishes a parent Linear issue (or rewrites the existing one in place when you passed a ticket ID):

- The **top** of the body is a terse, human-facing RFC: Problem / Background / Proposal / Constraints / Implementation / References. This is what teammates read.
- The **bottom** is a `PLAN.md` collapsible containing the full agent-facing brief: user stories, deeper implementation decisions, testing decisions. This is what Specular reads.

### `/specular:plan ABC-123`

Slices the parent into sub-issues. Each one is classified in its body (no Linear labels involved):

- **AFK** - safe to implement unattended. The implement loop will pick it up. Default - no marker in the body.
- **HITL** - needs you in the loop (design calls, risky migrations, anything ambiguous). Marked with a `**Type:** HITL` line in the body. The loop skips these.

### `/specular:implement ABC-123`

An orchestrator that drives subagents. It creates one worktree for the parent (a sibling of your CWD, named after Linear's `branchName`), installs dependencies once, then works the eligible AFK sub-issues in order. Per sub-issue:

1. An **implementer** subagent builds it test-first via `/specular:work-on-issue`, runs the project's lint/typecheck/test commands as a hard gate (auto-detected from `package.json`, `Cargo.toml`, `Makefile`, etc.), and commits. It does not push.
2. A **reviewer** subagent asks one question - does this commit deliver what the ticket asked for? The build is already green when it runs, so it's scoped to that, capped at three findings, and returns `NONE` for most slices.
3. A **fixer** subagent runs only when there were findings. It addresses exactly those, re-runs the full validation, and amends the commit.
4. The orchestrator pushes and transitions the sub-issue to **Done** via the Linear MCP.

Subagents commit; only the orchestrator pushes. That's what makes it safe for the fixer to amend. Once no eligible sub-issues remain, the orchestrator opens a draft PR. The first failure halts the run - later slices build on earlier ones, so there's nothing useful to do past a break.

## FAQ

**Why a separate `SPECULAR.md` instead of a section in `CLAUDE.md`?**

So Specular doesn't squat on shared real estate. `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` are used by every agent tool you might run; if you decide to swap Specular out for another speccing framework, deleting `SPECULAR.md` cleanly removes Specular's footprint. See [`SPECULAR.md`](#specularmd) for the full rationale.

**Why is the RFC format so short - I expected a full Shape Up pitch.**

Intentional. Shape Up pitches are meant to be argued over by humans before any work starts; Specular's RFC is meant to be skim-readable in a Linear sidebar so teammates actually read it. The verbose agent-facing material lives in the `PLAN.md` collapsible at the bottom of the issue body. See [RFC format](#rfc-format).

**The loop keeps stopping for permission prompts. What now?**

Re-run `/specular:setup` and tell it the binary that wasn't allowlisted (or just answer "yes" when it asks "anything else the loop is likely to run?"). The allowlist is meant to be iterated on - anything a subagent reaches for that isn't in `.claude/settings.local.json` will interrupt an otherwise unattended run.

**Can I use Specular without Linear?**

Not currently. Linear is the source of truth for issues, plans, and status transitions, and the implement loop pulls work from there.

**Does the implement loop run on my machine or in the cloud?**

On your machine, in your Claude Code session. There is no remote execution.

**What happens if I `Ctrl-C` mid-loop?**

The running subagent gets killed. Anything already pushed and marked **Done** stays, and re-running picks up from the first sub-issue that isn't Done. A slice that was committed but not yet pushed is still sitting in the worktree - the next run re-does it.

**Why does the reviewer barely ever find anything?**

By design. It runs after lint, typecheck, and tests have already passed, so it only checks whether the commit does what the ticket said. Style, refactors, naming, and speculative edge cases are explicitly out of scope, and it's capped at three findings. A reviewer that always finds something just adds a fixer round to every slice.

## References

- [mattpocock/skills](https://github.com/mattpocock/skills) - source for the TDD and module-design notes.
- [ghuntley.com/loop](https://ghuntley.com/loop) - source for the loop pattern this started as (the `ralph` loop).
- [basecamp.com/shapeup](https://basecamp.com/shapeup) - source for the RFC structure (compressed for skim-readability).
