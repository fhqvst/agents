---
name: specify
description: Spec out a change as a Linear RFC issue. Optionally grills the user first to sharpen domain language, then publishes a parent Linear issue with a terse human-facing RFC at the top and a comprehensive agent-facing PLAN.md collapsible at the bottom. Use when the user wants to spec out a change, write an RFC, or stress-test an idea against the existing domain model. Trigger on phrases like "spec this out", "write an RFC for X", "create an RFC", "stress-test this idea".
---

# Specify

Produce a Linear issue with two layers:

1. **Top (human-facing, terse):** Problem, optional Background, Proposal, Constraints, Implementation, References. Reviewers should be able to evaluate the direction in under a minute.
2. **Bottom (agent-facing, additive):** a single Linear `+++ PLAN.md` collapsible holding what the human-top doesn't cover. Downstream skills (`/specular:plan`, `/specular:implement`) read the entire issue body - the human-top plus `PLAN.md` together are the full brief. `PLAN.md` should not restate Problem, Proposal, or Out-of-scope. The one exception is Implementation: the human-top has a high-level pseudocode glance, and `PLAN.md` goes deeper.

## 0. Confirm Linear context

The repo's agent-instruction file (`CLAUDE.md` / `AGENTS.md` / `GEMINI.md`) should tell you which Linear team and project to file new issues under, and whether RFCs should carry a specific label or assignee. If that prose isn't there, tell the user to run `/specular:setup` first and stop.

## 1. Decide whether to grill

Pick one based on the user's opening message:

- **Grill mode** - the user is fuzzy ("I want to add notifications somewhere"), is using overloaded terms, or explicitly asks to be challenged / stress-tested. Run section 2.
- **Direct mode** - the user has a concrete spec in mind and just wants it written up. Skip to section 3.

If unsure, default to direct mode and offer: *"Want me to grill you on this first, or go straight to the RFC?"*

## 2. Grill (only in grill mode)

Interview the user relentlessly about every aspect of this plan until you reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time, waiting for feedback on each before continuing. If a question can be answered by exploring the codebase, explore the codebase instead.

### Anchor in the current UI (nice-to-have, requires Playwright MCP)

This step is opt-in based on tool availability. **If a Playwright MCP isn't installed in this session (no `mcp__plugin_playwright_playwright__browser_*` tools available), skip silently and grill from the codebase alone.** Don't ask the user to install anything.

If Playwright is available, decide whether this change ships any user-visible surface. Cues: the seed prompt mentions a URL, or words like *page, screen, view, button, drawer, modal, sheet, panel, flow, sidebar, header, layout*; the codebase area is clearly frontend; the user explicitly asks ("please look at the page first").

If yes:

1. Ask the user for the URL if it isn't in the seed prompt and you can't infer it from the agent-instruction file.
2. Navigate with `browser_navigate` and capture the relevant view(s) with `browser_take_screenshot`. Resize the viewport to a representative desktop size (1440x900) first.
3. Read the screenshot. Identify the layout primitives (rail, header, sidebar, table, etc.) before asking your first grilling question.

This grounds both you and the user in the same picture before grilling. The screenshots are ephemeral - they don't go in the spec.

Skip this entirely for non-UI changes (DB migrations, infra, internal libraries, etc.) - the inference cue is the absence of UI vocabulary in the seed prompt.

### Capturing vocabulary decisions

Capture vocabulary decisions in memory as you go. They land in `PLAN.md` under a **Vocabulary** section (see section 5). Use the same format as the persistent glossary - see [SPECULAR-FORMAT.md](./SPECULAR-FORMAT.md).

Don't write to disk during grilling. Vocabulary changes to the repo's `SPECULAR.md` files happen later: `/specular:plan` decides whether to schedule a sub-issue that updates them, and the implement loop performs the edits as part of normal sub-issue work.

Don't couple the vocabulary section to implementation details. Only include terms meaningful to domain experts.

### Grilling techniques

- **Explore the codebase first.** Get oriented before grilling. Read any nearby `SPECULAR.md` files (the project's hierarchical glossary) and any READMEs in the relevant module.
- **Sharpen fuzzy language.** Propose precise canonical terms. *"You're saying 'account' - do you mean the Customer or the User? Those are different things."*
- **Discuss concrete scenarios.** Stress-test relationships with specific scenarios that probe edge cases.
- **Cross-reference with code.** *"Your code cancels entire Orders, but you just said partial cancellation is possible - which is right?"*
- **Challenge against existing SPECULAR.md files.** If the relevant folder's `SPECULAR.md` (or any ancestor) defines existing domain language, call out conflicts when the user uses a term differently.

When the grilling is done, fall through to section 3.

## 3. Explore the codebase

If section 2 ran, you already explored the relevant area - skip this step except for files you specifically need to cite or to clarify a precise interface.

If section 2 was skipped, read the area that will change now. Assemble the relevant vocabulary from `SPECULAR.md` files (see [SPECULAR-FORMAT.md](./SPECULAR-FORMAT.md)). Respect any ADRs. Read files directly - do not spawn a subagent.

## 4. Sketch the modules

List the major modules to build or modify. Look for **deep modules**: small stable interfaces hiding substantial functionality, testable in isolation.

Show the user the proposed module breakdown and ask:

- Do these modules match your expectations?
- Which ones should have tests?

Iterate until they approve. Keep this lightweight.

## 5. Write `PLAN.md` (the agent-facing addendum)

`PLAN.md` holds what the human-top doesn't cover. Downstream skills read the entire issue body, so don't restate the Problem, Proposal, or Out-of-scope here - they'll get those from the top. The one place restatement is welcome is Implementation: the human-top has a brief pseudocode glance, and `PLAN.md` goes deeper.

Write `PLAN.md` first - the human-top in section 7 is derived from it.

Use this template (omit any section that has nothing useful to add):

<plan-template>

## User Stories

A long, numbered list of user stories in the format:

1. As an <actor>, I want a <feature>, so that <benefit>

Cover all aspects of the feature - this list should be extensive.

## Vocabulary

(Only include this section if grilling produced new domain terms or sharpened existing ones.)

The terms introduced or refined while specifying this change. Use the same format as a `SPECULAR.md` (see [SPECULAR-FORMAT.md](./SPECULAR-FORMAT.md)). `/specular:plan` reads this section to decide whether to schedule a sub-issue that propagates these terms into the appropriate `SPECULAR.md` files.

## Implementation Decisions

Deeper-than-the-human-top detail on how to build this. The human-top already has the headline pseudocode; this section is for everything that didn't fit there:

- Modules to build or modify (from section 4) and why each exists
- Full interfaces those modules expose (beyond the human-top's "// New" / "// Changed" glance)
- Architectural decisions and the tradeoffs behind them
- Schema changes
- API contracts
- Specific interactions between components

Do NOT include file paths or code snippets - they go stale fast. Pseudocode TypeScript signatures are fine for clarifying interfaces.

## Testing Decisions

- What makes a good test here (test external behavior, not implementation details)
- Which modules will be tested (from section 4)
- Prior art - similar tests already in the codebase to mirror

## Further Notes

Anything else worth capturing - non-obvious constraints, open questions, references to relevant files.

</plan-template>

Hold this content in memory - it goes into the `+++ PLAN.md` collapsible in section 8.

## 6. Build a low-fi UX prototype (nice-to-have, requires Figma MCP)

This step is opt-in based on tool availability. **If a Figma MCP isn't installed in this session (no `mcp__plugin_figma_figma__*` tools available), skip silently and proceed to section 7.** Don't ask the user to install anything. The spec is fine without a prototype - this is purely additive when Figma is around.

If Figma is available and the change ships any user-visible surface (use the same UI-detection cues as section 2), build a clickable wireframe so reviewers can react to the proposed UX in seconds instead of reading prose.

The wireframe is intentionally low-fi - gray rectangles only, mirroring the real app's shell so the proposed change lands in context. Skip pixel polish; this is a spec-time artifact, not a design deliverable.

1. **Create a file.** `mcp__plugin_figma_figma__create_new_file` with a descriptive name (e.g. *"<feature> wireframe - <project>"*). Get the user's `planKey` from `whoami` if you don't have it.
2. **Build at least two frames** with `use_figma`:
   - One **Closed** / current-state frame: the existing shell (rail, header, sidebar, table, etc.) as you saw it in section 2's screenshots.
   - One **Open** / proposed-state frame: the same shell with the change in place (drawer slid in, modal open, new section visible, etc.).
   - For multi-step flows, add intermediate frames as needed.
3. **Wire prototype reactions.** `ON_CLICK` → `SMART_ANIMATE` between frames so the prototype is actually clickable. Set `figma.currentPage.flowStartingPoints` to the starting frame.
4. **Verify visually.** `get_screenshot` on each frame. Fix layout issues before moving on.

Load the `figma:figma-use` skill before any `use_figma` call - it's a mandatory prerequisite.

The Figma file URL gets attached to the Linear issue as a link in section 9. The wireframe is referenced from the human-facing top's `## UX` section (see section 7).

Skip this entirely for non-UI changes.

## 7. Derive the human-facing top

Now write the terse top of the body, deriving from `PLAN.md`. Use this exact structure:

```
## Problem
[1-3 sentences. What's broken, missing, or painful. Make it concrete -
if there's a user-facing or developer-facing symptom, name it.]

## Background (optional)
[Only include this when the area would be unfamiliar to most reviewers.
A few sentences to orient someone who hasn't touched this part of the
codebase. Skip entirely if the area is well-known.]

## Proposal
[What we're doing and why. Written as a clear direction, not a menu of
options. Should read naturally - not like a template was filled in.

Then, if there are sensible alternatives reviewers might suggest,
preempt them: "Not doing X because Y." Only include alternatives a
reasonable engineer would actually propose. If the approach is
obvious, skip this part entirely.]

## Constraints

**In:**
- [Concrete things that change. Each bullet = one specific thing.]

**Out:**
- [Things we're explicitly not touching. Important because agents
  love to "improve" adjacent code.]

## Implementation

[Start with a single sentence describing the core implementation-level
change - the key thing that shifts in the code, not a restatement of
the proposal.

Then a pseudocode summary of interfaces being added or changed, inside
a fenced ts code block. Group into "// New" and "// Changed" sections.
Use short TypeScript-style signatures - just name, params, and return
type. Spread existing params with `...existing`.

Add a short inline `//` comment ONLY when the purpose isn't obvious
from the name and signature alone.

If some areas are too uncertain to spec, use bold sub-headers
(**Interfaces:** and **Open areas:**) to separate them.]

## UX (only if section 6 ran)

[Link to the Figma prototype with one sentence describing what it shows.
Format: `[Figma prototype](<url>) - <one sentence>`. The URL is also attached
to the Linear issue as a link in section 9, so this is a convenience for
in-body readers.]

## References
- [path/to/file](https://github.com/org/repo/blob/main/path/to/file)
- [path/to/other](https://github.com/org/repo/blob/main/path/to/other)
```

The References section links to 2-5 key files. Determine the org and repo from the git remote URL. Use the file path as the link text.

### Writing principles for the human-facing top

- **30-second rule**: an engineer should be able to read Problem, Background, and Proposal in under 30 seconds.
- **Implementation detail lives in its own section**: no code, interfaces, paths, or type signatures in Problem, Background, or Proposal.
- **No acceptance criteria**: don't write "should X when Y" checklists. The agent derives these from `PLAN.md`.
- **Concrete over abstract**: "stack traces point to our wrapper instead of the real throw site" beats "error reporting is suboptimal."
- **Lead with the pain, not the cause**: the first sentence of Problem should be the symptom.
- **Honest alternatives**: only mention approaches you genuinely considered and rejected for real reasons.
- **Constraints should be short and scannable**: one line per bullet.
- **One sentence per paragraph** in Problem, Background, and Proposal. Separate with double newlines so they render as distinct paragraphs in Linear.

## 8. Compose the issue body

Final body structure:

```markdown
<the human-facing top from section 7>

+++ PLAN.md

<the PLAN.md content from section 5>

+++
```

Linear collapsible syntax: `+++ <summary>` on its own line, blank line, content, blank line, closing `+++` on its own line. Don't use HTML `<details>` - Linear renders that as raw text.

## 9. Present and create

Show the composed body to the user. Ask if they want to adjust anything.

Once approved, create the issue with `mcp__plugin_linear_linear__save_issue`:

- Title: a concise summary derived from the Problem section
- Team / project / assignee / label: per the agent-instruction file's guidance. Resolve team and project names to IDs via `mcp__plugin_linear_linear__list_teams` / `list_projects`. If the file doesn't mention an assignee, leave it unassigned. If it doesn't mention a label, skip the label.
- Description: the composed body
- `links`: if section 6 ran, attach the Figma prototype URL as `[{url, title: "Figma prototype - <feature>"}]`. Linear renders this as a preview card in the issue sidebar.

Report the issue URL.
