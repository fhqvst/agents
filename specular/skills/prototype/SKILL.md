---
name: prototype
description: Build a clickable HTML prototype of a pitched Linear project as an Artifact, using the repo's real shadcn theme tokens and component class strings so it looks like the app. Reads the project's pitch and BRIEF.md, looks at the running app to calibrate, builds one board per place in the breadboard, verifies it in the browser, and links it from the project. Use when the user wants a prototype, mockup, or visual for a pitched project. Trigger on "prototype P-XXX-123", "mock this up", "make a visual for the pitch".
argument-hint: "<project-id>"
---

# Prototype

Turn a pitch into something people can click. The output is a single Artifact with one board per place in the pitch's breadboard, styled with the app's real design tokens and component markup, linked from the Linear project.

The breadboard is the spec. This skill adds no affordances, no places, and no copy the pitch didn't ask for. When the pitch is missing something the prototype needs, that is a finding to report, not a decision to make.

## 0. Confirm context

Find `SPECULAR.md` by walking upward from the current working directory. It needs:

- `## App` with `Dev` and `URL` bullets - to look at the current app.
- `## Design` with `Theme` and `Components` bullets, optionally `Fonts` - to build the kit.

If either section is missing, tell the user to run `/specular:setup` and stop.

## 1. Load the pitch

`$ARGUMENTS` is a Linear project reference. Fetch it with `mcp__plugin_linear_linear__get_project`.

The description has two parts written by `/specular:pitch`: the human top (Problem, Solution, Measuring success) and a `+++ BRIEF.md ... +++` collapsible. Read both. The top tells you what the change is for; everything the boards are built from is in `BRIEF.md`:

- **Breadboard**: places, affordances, connections. One board per place.
- **Copy**: use it verbatim.
- **Where the obvious approach is wrong** and **No-gos**: constraints on what the boards may show.
- **Look at**: the URLs to visit in section 2.
- **Vocabulary**: use these nouns in labels.

If there is no `+++ BRIEF.md` block, warn that the project wasn't written with `/specular:pitch` and ask the user for the list of places and the URL to look at before continuing.

## 2. Look at the app

Start the app with `mcp__Claude_Browser__preview_start` using the `specular-app` configuration (create `.claude/launch.json` next to `SPECULAR.md` from the `## App` section if it's missing - same shape as in `/specular:pitch`). Set the viewport to 1440x900.

Walk every entry in **Look at** and take a screenshot. If a screen needs a session, ask the user to log in in the browser pane and wait.

Read each screenshot and write down, for the shell and for each touched surface: the layout primitives, their proportions, spacing rhythm, the sidebar's items and active state, the header, the user menu, the footer. Use `mcp__Claude_Browser__read_page` to pull exact labels. This is the reference the prototype is calibrated against in section 5.

Screenshots come back inline and are not saved. The calibration happens by comparison, not by embedding them.

## 3. Build the kit

The kit is the CSS and markup vocabulary the boards are written in. It comes from the repo, not from memory.

1. **Theme.** Read the file at `Design → Theme`. It is a Tailwind v4 theme: `:root` and `.dark` variable blocks plus `@theme inline` mappings. Copy the variable blocks and the `@theme` blocks verbatim. Drop `@import` lines that point at packages (`tailwindcss`, `tw-animate-css`, `shadcn/tailwind.css`, generated typography) - the browser build provides Tailwind itself, and the typography file is read separately in step 3.
2. **Typography.** If the theme imports a generated typography CSS, read it and copy its `@theme` blocks too. Fonts are the one thing the kit may not copy freely: if `Design → Fonts` is set, the fonts there are open-licensed and you publish the regular, medium, and semibold cuts as supporting files with `@font-face` declarations under the family name the theme's `--font-sans` expects. If it is not set, the brand face is proprietary or unknown - never publish it. Pick the closest open stand-in from Google Fonts (a neutral grotesque for a grotesque, and so on), load it from the allowed fonts host, and name the substitution in the Prototype section's open questions so reviewers know the type is approximate.
3. **Components.** For each element the boards need, read the source under `Design → Components` and copy the class strings exactly. Typical set: the sidebar and app shell, page header, button variants, badge, table, dialog, input and input group, dropdown menu, card, empty state. Composite components in the library (app shell, page header, copy input) are read the same way - their JSX tells you the nesting and the classes. Keep `data-slot` attributes; some styles key off them.
4. **Icons.** The library uses Phosphor. Load nothing; inline the SVG path for the handful of icons the boards use, sized with the same `size-4` convention the components expect.
5. **Tailwind.** Load the v4 browser build from the allowed CDN and put the theme in a `<style type="text/tailwindcss">` block:

   ```html
   <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
   <style type="text/tailwindcss">
     @import "tailwindcss";
     @custom-variant dark (&:where(.dark, .dark *));
     /* theme variable blocks and @theme blocks, verbatim */
   </style>
   ```

6. **Dark mode.** The app toggles a `.dark` class on the root. Match the screenshot you took: if the app was dark, start dark. Add a small inline script that sets `.dark` from `data-theme` or `prefers-color-scheme` so the artifact respects the viewer's theme, and paint `body` with `bg-background text-foreground` so nothing borrows the host's colors.

Load the `artifact-design` skill before writing the page. It is mandatory for every Artifact.

## 4. Build the boards

One HTML file. Structure:

- A slim strip at the top listing the boards by name, in breadboard order, with the current one highlighted. Clicking a name shows that board.
- Below it, one board at a time: the full app shell at 1440px wide (sidebar, header, content), inside a container with horizontal scroll so it works on narrow screens without reflowing the design.
- The first board is **Current**: the screen as it is today, rebuilt with the kit. It exists to calibrate fidelity and to show reviewers the delta.
- Then one board per place in the breadboard, in order. Dialogs and confirms are their own boards, rendered over the page they open from.

Wire the connections: each affordance that the breadboard connects to a place becomes a click that switches to that board. Affordances with no connection do nothing. Do not add hover states, toasts, or transitions the pitch didn't mention.

Fill the boards with realistic data in the app's own vocabulary - real-looking symbols, plausible dates, the actual copy from `BRIEF.md`. Cover the states the pitch names: if it says "empty state", one board shows it.

Put the file in the scratchpad directory. Title it after the project name.

## 5. Verify against the app

Publish once with the `Artifact` tool (a favicon is required on first publish; pick one that fits the project and never change it on republish), then open the URL in the browser pane and screenshot the **Current** board at 1440x900.

Compare it to the screenshot of the real screen from section 2. Fix drift in this order: shell proportions, spacing rhythm, type scale, colors, then details. Republish and re-check until a reviewer would not notice the swap at a glance. Only then screenshot the other boards and check each connection by clicking it.

If a board cannot be made faithful because the pitch lacks a detail (a label, a state, an ordering), note it as a finding for section 6. Do not invent it silently.

## 6. Link it from the project

Update the Linear project with `mcp__plugin_linear_linear__save_project`:

- `links`: `[{url: "<artifact url>", title: "Prototype - <project name>"}]`. Links are append-only, so this never disturbs existing ones.
- `patch`: insert a `## Prototype` section directly before `## Measuring success`:

  ```markdown
  ## Prototype

  [Prototype](<artifact url>) - <one sentence: which boards exist and what to click>.

  **Open questions from prototyping:**
  - <one line per finding from section 5, or omit the list if there were none>

  ```

  Use `insert_before` with anchor `## Measuring success`. Touch nothing else in the description. If a `## Prototype` section already exists (a re-run), replace it with `replace_range` from `## Prototype` to `## Measuring success`.

Report the artifact URL, the boards it contains, and the open questions. The next step is `/specular:specify "<seed>"` for each line in the pitch's Suggested first cut.
