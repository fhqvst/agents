---
name: setup
description: Create or update the repo's `SPECULAR.md` (Linear routing + validation commands) and pre-approve the tools the headless implement loop needs in `.claude/settings.json`. Use when the user asks to "set up specular", configure specular, or when other specular skills report that this context is missing.
---

# Setup

Specular keeps its config in a single `SPECULAR.md` at the repo root. This skill creates or updates that file and the `.claude/settings.json` allowlist. It never touches `CLAUDE.md` / `AGENTS.md` / `GEMINI.md`.

This skill's job is to confirm:

1. Hard dependencies are installed (Linear MCP, `git`, `gh`, `jq`).
2. `SPECULAR.md` at the repo root names the Linear **team** and (optionally) **project** new issues should be filed under.
3. `SPECULAR.md` names the project's **lint**, **typecheck**, and **test** commands (when they exist).
4. The repo's `.claude/settings.json` pre-approves the tools the headless implement loop will need.

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

Look for `SPECULAR.md` at the repo root. If it doesn't exist, you'll create it later in this skill. The expected shape:

```md
# Specular

## Linear

File Linear tickets under team **<team>**, project **<project>**. Leave new tickets unassigned unless told otherwise.

## Validation

Run `<lint cmd>` for lint, `<typecheck cmd>` for typecheck, `<test cmd>` for tests.
```

Both sections are prose, not strict syntax - downstream skills read them with an LLM. Free-form additions (monorepo path → project routing, assignee policy, RFC label preferences, "no typechecker", etc.) are fine.

### 2. Linear routing

Does `SPECULAR.md` already have a `## Linear` section that names a team and project?

If missing:

- Use `mcp__plugin_linear_linear__list_teams` to list teams. Ask which one.
- Use `mcp__plugin_linear_linear__list_projects` (filtered by team) to list projects. Ask which one, or "none".
- For monorepos with multiple Linear projects, ask whether routing depends on the area being changed (e.g. `auth/*` → one project, `billing/*` → another). Capture the routing rule as prose under `## Linear`.
- Propose the section text, e.g.:

  > File Linear tickets under team **Platform**, project **Q2 Roadmap**. Leave new tickets unassigned unless told otherwise.

  (If the user wants tickets self-assigned by default, or a specific RFC label, capture that here too.)

- Show the diff and ask before writing.

### 3. Validation commands

Does `SPECULAR.md` already have a `## Validation` section?

If missing, detect candidates from the repo (don't ask blind):

- `package.json` `scripts` (`lint`, `typecheck`, `test`, `check`, `tsc`)
- `Cargo.toml` (Rust: `cargo clippy`, `cargo check`, `cargo test`)
- `pyproject.toml` / `setup.py` / `tox.ini` / `Makefile` / `justfile`
- `bun.lockb` / `pnpm-lock.yaml` / `yarn.lock` / `package-lock.json` for the runner

Propose concrete commands based on what you found, then ask the user: *"Does this look right? Anything I missed, or any commands I should drop?"* If a category doesn't apply (no typechecker, etc.), say so explicitly so downstream skills know to skip it. Example section:

> Run `bun run lint` for lint, `bun run typecheck` for typecheck, `bun test` for tests. No separate end-to-end suite.

`/specular:implement` will run these as a hard gate before each commit.

### 4. Build the `.claude/settings.json` allowlist

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
Bash(git push:*)
Bash(gh pr create:*)
Bash(gh pr view:*)
Bash(jq:*)
```

Always required - the loop commits, pushes, opens a PR, and parses JSON. Keep the list narrow: only the verbs the sub-skills actually invoke (see `create-commit/SKILL.md` and `create-pr/SKILL.md`). If a future sub-skill needs more (e.g. `git rebase`, `gh pr comment`), add it then.

#### c. MCP servers

- Linear is a hard dependency (already checked in step 0). Always include:
  ```
  mcp__plugin_linear_linear__*
  ```
- For other MCP servers, inspect `.mcp.json` (repo) and `~/.claude.json` (`mcpServers`). For each additional server installed, ask the user: *"The loop has access to <server>. Will sub-issues plausibly use it (testing, codegen, design refs, etc.)?"* If yes, add `mcp__<server-prefix>__*`. Common candidates:
  - `mcp__plugin_playwright_playwright__*` - browser testing
  - `mcp__plugin_figma_figma__*` - design references
  - other project-specific servers

#### d. Validation + anything else the loop will run

Seed the proposal from the validation commands captured in step 3. For each command, add a `Bash(<leading-binary>:*)` entry. Examples:

- `bun run lint`, `bun test` → `Bash(bun:*)`, `Bash(bunx:*)`
- `cargo clippy`, `cargo test` → `Bash(cargo:*)`
- `pnpm lint`, `pnpm test` → `Bash(pnpm:*)`, `Bash(pnpx:*)`
- `./scripts/check.sh` → `Bash(./scripts/check.sh:*)`

Then **ask the user**: *"Anything else the loop will run that isn't covered? Codegen, migrations, deploy CLIs, custom `scripts/`?"* - add what they mention as additional `Bash(...:*)` entries.

#### Merge and write

Read `.claude/settings.json` if it exists. Merge - don't clobber - any existing `permissions.allow` array. Deduplicate. Then:

- If the file doesn't exist, propose creating it with `{ "permissions": { "allow": [...] } }`.
- If it exists and already covers everything, say so and skip.
- Otherwise show a diff of just the new entries being added and ask before writing.

Mention to the user that this is the repo-level settings file and will be committed so their teammates share the allowlist. If they'd rather keep it user-level, point them at `~/.claude/settings.json` and let them paste it there instead. Also remind them: if the loop ever stalls on a permission prompt, re-run `/specular:setup` and add the missing entry - this is meant to be iterated on, not gotten perfect on the first pass.

### 5. Done

Write any pending changes to `SPECULAR.md` (showing a diff first) and tell the user setup is complete. `/specular:specify`, `/specular:plan`, and `/specular:implement` will pick up Linear routing and validation commands from `SPECULAR.md`.
