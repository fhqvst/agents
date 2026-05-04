---
name: setup
description: Make sure the repo's CLAUDE.md (or AGENTS.md / GEMINI.md) tells specular which Linear team and project to file issues under and which lint/typecheck/test commands to run, and that `.claude/settings.json` pre-approves the Bash patterns the headless implement loop will need. Use when the user asks to "set up specular", configure specular, or when other specular skills report that this context is missing.
---

# Setup

Specular reads everything it needs from the repo's agent-instruction file (`CLAUDE.md`, `AGENTS.md`, or `GEMINI.md` - whichever the project uses). There is no separate config file.

This skill's job is to confirm:

1. The agent-instruction file names the Linear **team** and (optionally) **project** new issues should be filed under.
2. The agent-instruction file names the project's **lint**, **typecheck**, and **test** commands.
3. The repo's `.claude/settings.json` pre-approves the Bash patterns the headless implement loop will need (`git`, `gh`, `jq`, `claude`, plus whatever the validation commands invoke).

If anything is missing, offer to add it.

## Process

### 0. Verify the Linear MCP is available

Try a no-op call: `mcp__plugin_linear_linear__list_teams`.

If the tool is not available, the Linear plugin isn't installed. Tell the user:

> The Linear MCP plugin isn't installed. Install it from the Claude Code plugin marketplace (`/plugin` in the Claude Code UI, or follow https://docs.claude.com/en/docs/claude-code/plugins) and re-run `/specular:setup`.

Then stop. Do not continue without the MCP.

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

### 4. Check Bash permission allowlist

The implement loop runs headless (`claude --print`) and cannot surface permission prompts mid-flight - any Bash invocation that isn't pre-approved will stall it. The lint/typecheck/test trio from step 3 is **not** the full set: the loop will also run build steps, codegen, migrations, formatters, ad-hoc scripts, file ops, and whatever else a sub-issue plausibly requires. Be generous - false positives here are cheap, false negatives stall the loop.

Make sure `.claude/settings.json` (repo-level, committed so the team shares the allowlist) has a `permissions.allow` entry covering everything the loop is likely to invoke.

Build the proposed allowlist by **detecting manifests, not by parsing the validation commands**:

- **Baseline (always include):** `Bash(git *)`, `Bash(gh *)`, `Bash(jq *)`, `Bash(claude *)`, `Bash(specular-ralph *)`, `Bash(mkdir *)`, `Bash(rm *)`, `Bash(mv *)`, `Bash(cp *)`, `Bash(ls *)`, `Bash(cat *)`, `Bash(grep *)`, `Bash(rg *)`, `Bash(ag *)`, `Bash(find *)`, `Bash(sed *)`, `Bash(awk *)`, `Bash(echo *)`, `Bash(test *)`, `Bash(which *)`, `Bash(env *)`, `Bash(printenv *)`.
- **JS/TS** - if `package.json` exists, add `Bash(node *)`, `Bash(npx *)`, `Bash(tsc *)`, `Bash(eslint *)`, `Bash(prettier *)`, `Bash(vitest *)`, `Bash(jest *)`, plus the runner indicated by the lockfile:
  - `bun.lockb` / `bun.lock` → `Bash(bun *)`, `Bash(bunx *)`
  - `pnpm-lock.yaml` → `Bash(pnpm *)`, `Bash(pnpx *)`
  - `yarn.lock` → `Bash(yarn *)`
  - `package-lock.json` → `Bash(npm *)`
- **Rust** - if `Cargo.toml` exists, add `Bash(cargo *)`, `Bash(rustc *)`, `Bash(rustup *)`.
- **Python** - if `pyproject.toml`, `setup.py`, `requirements.txt`, or `tox.ini` exists, add `Bash(python *)`, `Bash(python3 *)`, `Bash(pip *)`, `Bash(pip3 *)`, `Bash(pytest *)`, `Bash(ruff *)`, `Bash(mypy *)`, `Bash(black *)`, `Bash(uv *)`, `Bash(poetry *)`, `Bash(tox *)`.
- **Go** - if `go.mod` exists, add `Bash(go *)`, `Bash(gofmt *)`.
- **Ruby** - if `Gemfile` exists, add `Bash(ruby *)`, `Bash(bundle *)`, `Bash(rake *)`, `Bash(rspec *)`.
- **Task runners** - if `Makefile` exists add `Bash(make *)`; if `justfile` / `Justfile` exists add `Bash(just *)`; if `Taskfile.yml` exists add `Bash(task *)`.
- **From the validation commands** confirmed in step 3 - cover the leading binary of each, in case it isn't a runner already pulled in by a manifest above (e.g. `swiftlint`, custom `./scripts/check.sh` → `Bash(./scripts/check.sh *)`).
- **Scan `package.json` `scripts`, `Makefile` targets, and any `justfile`** for additional binaries the project actually uses (e.g. `prisma`, `drizzle-kit`, `vite`, `next`, `playwright`, `storybook`, `wrangler`, `terraform`, `docker`, `docker-compose`). Add a `Bash(<binary> *)` entry for each one you spot.

Then **ask the user**: "Anything else the loop is likely to run that isn't on this list?" - codegen tools, migration runners, deployment CLIs, custom scripts under `scripts/` or `bin/`, etc. Add what they mention.

Read `.claude/settings.json` if it exists. Merge - don't clobber - any existing `permissions.allow` array. Deduplicate. Then:

- If the file doesn't exist, propose creating it with `{ "permissions": { "allow": [...] } }`.
- If it exists and already covers everything, say so and skip.
- Otherwise show a diff of just the new entries being added and ask before writing.

Mention to the user that this is the repo-level settings file and will be committed so their teammates share the allowlist. If they'd rather keep it user-level, point them at `~/.claude/settings.json` and let them paste it there instead. Also remind them: if the loop ever stalls on a permission prompt, re-run `/specular:setup` and add the missing binary to the list - this is meant to be iterated on, not gotten perfect on the first pass.

### 5. Done

Tell the user setup is complete and that `/specular:specify`, `/specular:plan`, and `/specular:implement` will pick up everything they need from the agent-instruction file.
