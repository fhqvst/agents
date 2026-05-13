---
name: setup
description: Make sure the repo's CLAUDE.md (or AGENTS.md / GEMINI.md) tells specular which Linear team and project to file issues under and which lint/typecheck/test commands to run, and that `.claude/settings.json` pre-approves the tools the headless implement loop will need. Use when the user asks to "set up specular", configure specular, or when other specular skills report that this context is missing.
---

# Setup

Specular reads everything it needs from the repo's agent-instruction file (`CLAUDE.md`, `AGENTS.md`, or `GEMINI.md` - whichever the project uses). There is no separate config file.

This skill's job is to confirm:

1. Hard dependencies are installed (Linear MCP, `git`, `gh`, `jq`).
2. The agent-instruction file names the Linear **team** and (optionally) **project** new issues should be filed under.
3. The agent-instruction file names the project's **lint**, **typecheck**, and **test** commands.
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

### 1. Find the agent-instruction file

Look for `CLAUDE.md`, `AGENTS.md`, or `GEMINI.md` at the repo root, in that order. If none exists, ask which one the project wants to use and create it empty.

### 2. Check Linear context

Read the file. Does it tell an agent which Linear team (and optionally project) to file new tickets under? Look for any prose along the lines of *"File Linear tickets under team X, project Y"*.

If missing:

- Use `mcp__plugin_linear_linear__list_teams` to list teams. Ask which one.
- Use `mcp__plugin_linear_linear__list_projects` (filtered by team) to list projects. Ask which one, or "none".
- Propose a one-line addition to the agent-instruction file, e.g.:

  > File Linear tickets under team **Platform**, project **Q2 Roadmap**. Leave new tickets unassigned unless told otherwise.

  (If the user wants tickets self-assigned by default, mention that instead. If they want a specific RFC label applied, mention that too.)

- Show the diff and ask before writing.

### 3. Check validation commands

Does the file tell an agent how to lint, typecheck, and test? Look for prose like *"Run `bun run lint` to lint, `bun run typecheck` to typecheck, `bun test` to run tests."*

If missing, detect candidates from the repo (don't ask blind):

- `package.json` `scripts` (`lint`, `typecheck`, `test`, `check`, `tsc`)
- `Cargo.toml` (Rust: `cargo clippy`, `cargo check`, `cargo test`)
- `pyproject.toml` / `setup.py` / `tox.ini` / `Makefile` / `justfile`
- `bun.lockb` / `pnpm-lock.yaml` / `yarn.lock` / `package-lock.json` for the runner

Propose concrete commands and let the user confirm or edit. If a category doesn't apply (no typechecker, etc.), say so explicitly so downstream skills know to skip it. Example addition:

> Validation commands: `bun run lint` for lint, `bun run typecheck` for typecheck, `bun test` for tests. No separate end-to-end suite.

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

#### d. Validation / build commands

Derive **only** from the lint/typecheck/test commands confirmed in step 3, plus their runner. For each command, add a `Bash(<leading-binary> *)` entry. Examples:

- `bun run lint`, `bun run typecheck`, `bun test` → `Bash(bun *)`, `Bash(bunx *)`
- `cargo clippy`, `cargo test` → `Bash(cargo *)`
- `pnpm lint`, `pnpm test` → `Bash(pnpm *)`, `Bash(pnpx *)`
- `./scripts/check.sh` → `Bash(./scripts/check.sh *)`

Skip this category entirely if step 3 declared no validation commands.

Then **ask the user**: *"Anything else the loop will run that isn't covered? Codegen, migrations, deploy CLIs, custom `scripts/`?"* - add what they mention as additional `Bash(...)` entries.

#### Merge and write

Read `.claude/settings.json` if it exists. Merge - don't clobber - any existing `permissions.allow` array. Deduplicate. Then:

- If the file doesn't exist, propose creating it with `{ "permissions": { "allow": [...] } }`.
- If it exists and already covers everything, say so and skip.
- Otherwise show a diff of just the new entries being added and ask before writing.

Mention to the user that this is the repo-level settings file and will be committed so their teammates share the allowlist. If they'd rather keep it user-level, point them at `~/.claude/settings.json` and let them paste it there instead. Also remind them: if the loop ever stalls on a permission prompt, re-run `/specular:setup` and add the missing entry - this is meant to be iterated on, not gotten perfect on the first pass.

### 5. Done

Tell the user setup is complete and that `/specular:specify`, `/specular:plan`, and `/specular:implement` will pick up everything they need from the agent-instruction file.
