---
name: pitch
description: Shape a project as a Linear project description with two layers - a one-minute human pitch (Problem, Solution, Measuring success) and an agent-facing BRIEF.md collapsible holding the breadboard, copy, forks, no-gos, decisions, and scenarios that /specular:prototype and /specular:specify read later. Looks at the running app first when the work touches UI. Use when the user wants to shape a project, write a pitch, or turn a rough idea into a Linear project. Trigger on "pitch this", "shape X", "write a project description for X".
argument-hint: "<short description | project-id>"
---

# Pitch

Produce a Linear **project** whose description has two layers:

1. **Top (human-facing, one minute):** Problem, Solution, Measuring success. A reviewer should be able to say yes or no to the direction without opening anything else. `/specular:prototype` adds a Prototype section between Solution and Measuring success later.
2. **Bottom (agent-facing, additive):** a single Linear `+++ BRIEF.md` collapsible holding everything the next agent needs and the reviewer doesn't: the breadboard, copy, forks, no-gos, decisions, scenarios. `/specular:prototype` and `/specular:specify` read the whole description - top plus `BRIEF.md` is the full brief. `BRIEF.md` never restates Problem or Solution.

The split is by reader, not by importance. If a line helps an agent build the right thing but a reviewer doesn't need it to judge the direction, it goes in `BRIEF.md`.

A pitch sits one level above an RFC. It describes places and affordances, never interfaces, files, or estimates. Implementation is done by agents, so there is no appetite or time box: the constraint on scope is what can be verified, and that lives in **Measuring success**.

The pipeline is `pitch` → `prototype` → `specify` (one RFC per item in the first cut) → `plan` → `implement`.

## 0. Confirm context

`SPECULAR.md` is found by walking upward from the current working directory until you hit it or reach `$HOME` / the filesystem root. It must have a `## Linear` section with at least one `### <project>` subsection. If not, tell the user to run `/specular:setup` first and stop.

If the pitch touches a user-visible surface (see section 2), `SPECULAR.md` also needs an `## App` section with `Dev` and `URL` bullets. If it's missing, tell the user to run `/specular:setup` and stop - looking at the app is not optional for UI work.

## 1. Load the input

`$ARGUMENTS` is either a short description or a Linear project reference (name, ID, `P-XXX-123`, or slug).

- **Description:** this is the seed. You will create a new project in section 7.
- **Project reference:** fetch it with `mcp__plugin_linear_linear__get_project`. Its current description, summary, and links are the seed - read them, keep anything the user has already decided, and rewrite the description in place in section 7. Never drop existing links.

## 2. Look at the app

Decide whether the pitch ships any user-visible surface. Cues: the seed mentions a URL or words like *page, screen, view, button, drawer, modal, sheet, panel, flow, sidebar, header, landing, form*; the area is clearly frontend. Non-UI pitches (migrations, pipelines, services) skip this section.

For UI pitches this step is mandatory. You cannot shape a change to a screen you haven't seen.

1. Start the app with `mcp__Claude_Browser__preview_start` using the `specular-app` configuration. If `.claude/launch.json` next to `SPECULAR.md` is missing, create it from the `## App` section first:

   ```json
   {
     "version": "0.0.1",
     "configurations": [
       {
         "name": "specular-app",
         "runtimeExecutable": "bash",
         "runtimeArgs": ["-c", "<Dev command from SPECULAR.md>"],
         "port": <port from URL>
       }
     ]
   }
   ```

2. Set the viewport to 1440x900 with `mcp__Claude_Browser__resize_window`.
3. Navigate to every screen the pitch touches and take a screenshot of each. If a screen needs a session, ask the user to log in in the browser pane and wait - the session persists for the rest of the run.
4. Read each screenshot and name the layout primitives you see (rail, header, sidebar, table, card, dialog) before asking your first question. Use `mcp__Claude_Browser__read_page` when you need exact labels or copy.
5. Keep a list of every URL and state you visited. It becomes the **Look at** section of `BRIEF.md`, which is how `/specular:prototype` regains this context in a fresh session.

Screenshots are not uploaded anywhere. The Look at list plus the prototype are the durable record.

## 3. Find the number

If the seed implies a metric (dropoff, conversion, time to first call, views), find the current value before grilling. Check for a PostHog MCP; if available, locate the insight and record the value and its link. If no tool can reach the number, ask the user for it once. If nobody has it, write "no baseline yet" in **Measuring success** and make establishing one the first item of the Suggested first cut.

Never invent a number.

## 4. Grill at breadboard altitude

Interview the user until you can write the breadboard with no guesses in it. Ask one question at a time, with your recommended answer. If the codebase or the app can answer a question, look instead of asking.

Stay at the right altitude:

- **Ask about:** actors (logged-out, free, paid, admin), places, affordances, connections, states (empty, loading, error, gated), copy for the key strings, what is out, how success is verified, and every fork where the plausible default is wrong.
- **Never ask about:** interfaces, modules, file paths, schema, estimates, or time. Those belong to `/specular:specify`.

### Techniques

- **Explore first.** Read the code behind each screen you looked at so questions are grounded in how it works today, not how it looks.
- **Sharpen fuzzy nouns.** *"You said 'account' - the Customer or the User? Keys belong to the Customer."*
- **Walk scenarios.** *"A paid customer rotates a key while an integration is live. What happens to the old key?"* Each scenario either confirms a decision or exposes a fork.
- **Hunt the wrong default.** For every affordance ask what an agent would build if told nothing more. If that is wrong, it goes in `BRIEF.md` under **Where the obvious approach is wrong**, with the chosen path.
- **Cut scope over machinery.** When a requirement needs new infrastructure, offer dropping the requirement before designing the infrastructure.
- **Record every answer.** Each one is a line in the Decisions log. Answers that live only in chat are the main thing this skill exists to prevent.

Stop when the breadboard has no open affordances and every scenario resolves.

## 5. Write `BRIEF.md` (the agent-facing addendum)

Write this first; the human top is derived from it. Omit any section with nothing to add.

<brief-template>

## Breadboard

A fenced text block:

```
Places:       <every screen, dialog, or panel involved>
<Place>:      <its affordances, comma-separated>
Connections:  <affordance> -> <place>; ...
```

Places and affordances, never columns, components, or pixels. `/specular:prototype` builds one board per place.

## Copy

The key strings: titles, empty states, the one error message that matters, CTA labels. Actual words, not descriptions of words.

## Where the obvious approach is wrong

One line per fork: the default an implementer would pick, and the path we chose instead. Only forks you have evidence for.

## No-gos

What is explicitly out. Adjacent things an eager implementer will want to touch.

## Vocabulary

Canonical nouns and what they are not. One line each. *"Key: an entitlements-service API key, owned by a Customer, never by a User."*

## Current state

How the touched surfaces work today, by module name with GitHub links to the entry points. Which systems the data comes from and goes to. No file paths in prose; links only.

## Decisions

Every answer from grilling, as a flat list. Each line is a fact a colleague or agent would otherwise re-ask. Include the ones that feel obvious, and the flag the work ships behind.

## Scenarios

5-10 concrete walkthroughs, numbered. Actor, starting state, steps, expected outcome. These are where the forks came from, and they are what `/specular:specify` turns into user stories.

## Look at

The URLs and states visited in section 2, one per line, with what to notice on each. Include the login requirement if any. `/specular:prototype` walks this list.

## Suggested first cut

One line per item of the Solution, written as the seed prompt to pass to `/specular:specify`, each followed by a two-line demo script: what someone opens and what they should see. Expected to change once work starts; this is a map, not a plan.

## References

2-5 links: prior projects, dashboards, docs, conversations.

</brief-template>

Hold this content in memory - it goes into the `+++ BRIEF.md` collapsible in section 7.

## 6. Derive the human-facing top

Use this exact structure:

```
## Problem
[A specific story with the number in it. Who hits this, what they do, what
happens instead. 1-3 sentences. Link the insight if there is one.]

## Solution
[What changes, in the reader's words: which places are touched and what
someone can do there afterwards that they can't today. 2-4 sentences.

Then, only if a reasonable reviewer would suggest it, one line per
alternative: "Not doing X because Y."]

## Measuring success
[The metric, its current value, the target. If there is no metric, the one
thing a reviewer opens to call it done.]
```

### Writing principles

- **One-minute rule:** the whole top reads in under a minute, about 150 words.
- **Reader split:** breadboards, copy, flags, forks, and no-gos live in `BRIEF.md`. If the top mentions a component, a string, or a default, move it down.
- **Concrete over abstract:** "22% of people who see the onboarding form leave without submitting" beats "onboarding has friction".
- **Lead with the pain:** the first sentence of Problem is the symptom.
- **One sentence per paragraph** in Problem, separated by blank lines so Linear renders them apart.
- **No time, no appetite, no estimates** anywhere in the description.
- **Humanize:** if a humanizer skill is available, run the top through it in embedded mode before showing it to the user. Keep the headings, the numbers, and the links.

## 7. Compose and save

Final description:

```markdown
<the human-facing top from section 6>

+++ BRIEF.md

<the BRIEF.md content from section 5>

+++
```

Linear collapsible syntax: `+++ <summary>` on its own line, blank line, content, blank line, closing `+++` on its own line. Don't use HTML `<details>`.

Show the composed description to the user and ask if they want to adjust anything.

Once approved, save with `mcp__plugin_linear_linear__save_project`:

- **New project:** `name` derived from the Solution's first sentence; `addTeams` set to the team of the `SPECULAR.md` catch-all project (resolve it with `get_project` on that project); `description` as composed; `summary` as the Problem's first sentence, trimmed to 255 characters. No labels, no milestones, no dates.
- **Existing project:** `id` plus `description`. Keep `links` untouched - they are append-only anyway.

Report the project URL, then the next two commands: `/specular:prototype <project-id>` for a visual to react to, and `/specular:specify "<seed>"` for each line in the Suggested first cut.
