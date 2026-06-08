---
name: setup
description: Create or update the repo's `SPECULAR.md` (Linear routing) and pre-approve the tools the headless implement loop needs in `.claude/settings.local.json`. Use when the user asks to "set up specular", configure specular, or when other specular skills report that this context is missing.
---

# Setup

Specular keeps its config in a single `SPECULAR.md` file. This skill creates or updates that file and the `.claude/settings.local.json` allowlist.

`SPECULAR.md` lives wherever the user runs `/specular:setup` from - by default the current working directory. That makes it work both for "file in repo root" and for parent-folder worktree layouts like `~/Development/foo/{main,branch-1,...}` where the user invokes Claude from `~/Development/foo` and wants every sibling worktree to inherit the same config. The file is found at read time by walking upward from CWD (see step 1).

This skill's job is to confirm:

1. Hard dependencies are installed (Linear MCP, `git`, `gh`, `jq`).
2. `SPECULAR.md` lists the Linear projects new issues can land in, plus any glob-based path routing for monorepos.
3. The repo's `.claude/settings.local.json` pre-approves the tools the headless implement loop will need.

If anything is missing, offer to add it.

## Process

### 0. Dependency check

The implement loop hard-requires:

- **Linear MCP plugin** - probe with `mcp__plugin_linear_linear__list_teams`.
- **`git`**, **`gh`**, **`jq`** binaries on PATH - check with `command -v git`, `command -v gh`, `command -v jq`.

If any are missing, stop and tell the user exactly what's missing and how to install it:

- Linear MCP: install from the Claude Code plugin marketplace (`/plugin` in the UI, or https://docs.claude.com/en/docs/claude-code/plugins).
- `git` / `gh` / `jq`: install via the user's package manager (e.g. `brew install gh jq`).

Do not continue until all four are available.

### 1. Read or create `SPECULAR.md`

To find an existing `SPECULAR.md`, start at the current working directory and walk upward, checking each directory. Stop at the first hit, or when you reach `$HOME` or the filesystem root - whichever comes first. If found, you'll update that file in place at the end of this skill.

If none is found, you'll create a new `SPECULAR.md` in the **current working directory** at the end of this skill (not the repo root, not a parent - exactly where the user invoked the skill from). Show the absolute path before writing so it's obvious where it landed.

The expected shape is a `## Linear` section with one `### <project>` subsection per Linear project this repo files into:

```md
# Specular

## Linear

### Q2 Roadmap (`<project-id>`)

- Assignee: filip (`<user-id>`)

### Web (`<project-id>`)

- Paths: `apps/web/**`, `packages/web-ui/**`
- Assignee: alice (`<user-id>`)
```

Rules:

- One `###` subsection per project; the project's name and id go in the heading.
- Each subsection has flat key-value bullets - never deeper than one level.
- `Paths` is a comma-separated list of globs. The project that has **no `Paths`** is the implicit default (the catch-all for anything that doesn't match a glob).
- `Assignee` is optional.
- Team is **not** stored - it's derivable from the project. Edge case: if the user wants "file under team X with no project", omit all `###` subsections and put flat `Team:` and (optional) `Assignee:` bullets directly under `## Linear` instead.
- Don't invent extra keys.

### 2. Linear projects

Does `SPECULAR.md` already have a `## Linear` section with at least one `### <project>` subsection (or a `Team:` line)? If yes, skip to step 3.

If missing, walk the user through projects **one at a time**. Never write a value the user hasn't explicitly approved - inferences from git author, repo name, or "who leads what" are too brittle to commit to disk without confirmation.

For each project:

- Call `mcp__plugin_linear_linear__list_projects`. Recommend the project that best matches the repo (by name overlap with the repo directory, recent commits, or `CLAUDE.md` context) but always show the full list. Ask: *"Which Linear project should new tickets be filed under? Recommend `<X>` - alternatives: `<Y>`, `<Z>`. Pick one, or say 'none' for team-only routing."*
- Ask about assignee for this project: *"Default assignee for tickets in `<project>`? Self-assigned, someone specific, or unassigned? Recommend unassigned."* Resolve via `mcp__plugin_linear_linear__list_users` if needed.
- After the first project is captured, ask: *"Any more Linear projects this repo files tickets into?"* and loop. Stop when they say no.

If they answered "none" (team-only routing), prompt for the team via `mcp__plugin_linear_linear__list_teams` and the assignee, then write flat `Team:` and (optional) `Assignee:` bullets under `## Linear` - skip the rest of this step and step 3.

Compose the `### <project>` subsections (without `Paths` for now), show a diff, and ask one final time before writing.

### 3. Path-based routing (monorepos)

Only relevant when step 2 produced **more than one** project subsection. If there's just one, skip - it's already the implicit default.

For each project except the catch-all, ask the user which paths it owns:

- *"Which paths or globs are covered by `<project>`?"* (plural - recommend concrete candidates based on the repo layout, e.g. `apps/web/**` and `packages/web-ui/**`, but never auto-pick).
- Add a `Paths: <glob>, <glob>` bullet to that project's subsection.

Leave exactly one project with no `Paths` bullet - it's the catch-all. If the user is unsure which one should be the catch-all, ask them.

`/specular:specify` matches RFC scope against `Paths` globs at file time. If nothing matches, it falls back to the project with no `Paths`.

### 4. Build the `.claude/settings.local.json` allowlist

The implement loop runs headless (`claude --print`) and cannot surface permission prompts mid-flight - any tool call that isn't pre-approved stalls it. The allowlist is **focused, not exhaustive** - it covers the categories below and nothing else. Read-only Bash globs auto-approve since 2.1.111, so generic file utilities (`cat`, `head`, `grep`, `find`, `ls`, etc.) don't belong here.

Build the proposed list from these four categories:

#### a. Specular's own sub-skill invocations

```
Skill(specular:*)
```

Lets the implement loop call `/specular:work-on-issue`, `/specular:create-commit`, `/specular:create-pr` etc. without prompting. (The driver `bin/specular-ralph` is gated by inline `allowed-tools` in `implement/SKILL.md` and does not need an entry here.)

#### b. Git + PR flow

```
Bash(git add:*)
Bash(git commit:*)
Bash(git status:*)
Bash(git diff:*)
Bash(git log:*)
Bash(git show:*)
Bash(git push:*)
Bash(git checkout:*)
Bash(git restore:*)
Bash(git clean:*)
Bash(git rm:*)
Bash(git worktree:*)
Bash(gh pr create:*)
Bash(gh pr view:*)
Bash(jq:*)
```

Always required - the loop commits, pushes, opens a PR, parses JSON, creates/enters its worktree, and cleans up stray changes between iterations (`git restore`/`clean`/`rm`, `git checkout`, `git show` to recall prior code). Keep the list narrow: only the verbs the sub-skills actually invoke (see `create-commit/SKILL.md` and `create-pr/SKILL.md`). If a future sub-skill needs more (e.g. `git rebase`, `gh pr comment`), add it then.

#### c. MCP servers

- Linear is a hard dependency (already checked in step 0). Always include:
  ```
  mcp__plugin_linear_linear__*
  ```
- For other MCP servers, inspect `.mcp.json` (repo) and `~/.claude.json` (`mcpServers`). For each additional server installed, ask the user: *"The loop has access to <server>. Will sub-issues plausibly use it (testing, codegen, design refs, etc.)?"* If yes, add `mcp__<server-prefix>__*`. Common candidates:
  - `mcp__plugin_playwright_playwright__*` - browser testing
  - `mcp__plugin_figma_figma__*` - design references
  - other project-specific servers

#### d. Project runners and anything else the loop will run

The implement loop figures out its own lint/typecheck/test commands per sub-issue (from `package.json`, `Cargo.toml`, `Makefile`, etc.), but the runners themselves still need to be pre-approved. Detect candidates from the repo:

- `bun.lockb` / `package.json` → `Bash(bun:*)`, `Bash(bunx:*)`
- `pnpm-lock.yaml` → `Bash(pnpm:*)`, `Bash(pnpx:*)`
- `yarn.lock` → `Bash(yarn:*)`
- `package-lock.json` → `Bash(npm:*)`, `Bash(npx:*)`
- `Cargo.toml` → `Bash(cargo:*)`
- `pyproject.toml` / `setup.py` → `Bash(uv:*)`, `Bash(python:*)`, `Bash(pytest:*)` as applicable
- `Makefile` / `justfile` → `Bash(make:*)`, `Bash(just:*)`

Propose the entries based on what you found. Then **ask the user**: *"Anything else the loop will run that isn't covered? Codegen, migrations, deploy CLIs, custom `scripts/`?"* - add what they mention as additional `Bash(...:*)` entries.

#### Merge and write

Read `.claude/settings.local.json` if it exists. Merge - don't clobber - any existing `permissions.allow` array. Deduplicate. Then:

- If the file doesn't exist, propose creating it with `{ "permissions": { "allow": [...] } }`.
- If it exists and already covers everything, say so and skip.
- Otherwise show a diff of just the new entries being added and ask before writing.

Mention to the user that this is the project-local settings file - it's gitignored and personal to them, so it won't be committed or shared with teammates. If they'd rather share the allowlist with the whole team, point them at `.claude/settings.json` (repo-level, committed); if they'd rather apply it across all their projects, point them at `~/.claude/settings.json` - and let them paste it there instead. Also remind them: if the loop ever stalls on a permission prompt, re-run `/specular:setup` and add the missing entry - this is meant to be iterated on, not gotten perfect on the first pass.

### 5. Done

Write any pending changes to `SPECULAR.md` (showing a diff and the absolute path first) and tell the user setup is complete. `/specular:specify`, `/specular:plan`, and `/specular:implement` will pick up Linear projects and path routing from `SPECULAR.md` by walking upward from CWD.
