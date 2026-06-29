---
name: study
description: Study a curriculum - an ordered path through many concepts, where each concept is taught by the /teach skill. Picks what to learn next, tracks progress across sessions, and creates a new curriculum on the fly when one doesn't exist yet.
disable-model-invocation: true
argument-hint: "[curriculum topic, or blank to resume]"
---

`/study` runs the user's personal **curriculum**: an ordered path through many concepts, where each concept is taught by the `/teach` skill.

Keep the two roles distinct:

- `/teach` teaches **one concept** well, inside one teaching workspace.
- `/study` decides **which concept comes next** and keeps the long arc of progress across many concepts and sessions.

## State

Just like `/teach`, all state lives in **the current directory** - treat it as the study root. Each curriculum is a subdirectory:

```
./                            # current directory = study root
  ACTIVE                      # single line: slug of the active curriculum
  <curriculum-slug>/          # one teaching workspace per curriculum
    CURRICULUM.md             # the syllabus + progress (see CURRICULUM-FORMAT.md)
    MISSION.md                # why the user is studying this (teach's MISSION format)
    RESOURCES.md
    NOTES.md
    reference/  learning-records/  lessons/  assets/
```

A curriculum directory **is** a `/teach` workspace, plus a `CURRICULUM.md` that sequences the concepts. So every file `/teach` expects already lives in the right place - `/study` just adds the syllabus on top and chooses the order.

## Flow

### 1. Resolve the active curriculum

Read `./ACTIVE` (if present) and list the curriculum directories in the current directory. Then branch:

- **The argument names a topic** (e.g. `/study product engineering concepts`):
  - It matches an existing curriculum → make that one active.
  - No match → **create a new curriculum** (§2) for that topic and make it active.
- **No argument:**
  - `ACTIVE` is set → resume it.
  - Curricula exist but none is active → list them with their progress (done/total) and ask which to study.
  - No curricula exist at all → ask the user what they want to study, then create it (§2).

Whenever the active curriculum changes, write its slug to `./ACTIVE`.

### 2. Create a curriculum (only when it doesn't exist)

Designing the **syllabus** is the one thing `/teach` doesn't do. Do it carefully:

1. Make `./<slug>/`.
2. **Establish the mission.** Interview the user on _why_ they want this (use teach's [MISSION-FORMAT](../teach/MISSION-FORMAT.md)). Don't skip this - the whole syllabus is grounded in it. Write `MISSION.md`.
3. **Find trusted resources first.** Never trust parametric knowledge for sequencing. Search for high-quality sources and seed `RESOURCES.md` (use teach's [RESOURCES-FORMAT](../teach/RESOURCES-FORMAT.md)).
4. **Design the ordered list of concepts** that carries the user from where they are to the mission. Each concept must be small enough to be a single `/teach` lesson, and ordered so each builds on the last. Write `CURRICULUM.md` per [CURRICULUM-FORMAT.md](./CURRICULUM-FORMAT.md).
5. **Show the user the syllabus** and let them reorder, add, or cut modules before you start teaching.

### 3. Teach the next concept

1. **Pick it.** Take the first not-done module in `CURRICULUM.md`, unless the `learning-records/` suggest the user is ready to skip ahead or needs to revisit something earlier. Confirm the pick in one line.
2. **Mark it in-progress** (`[>]`) in `CURRICULUM.md`.
3. **Delegate to `/teach`.** Read `${CLAUDE_PLUGIN_ROOT}/skills/teach/SKILL.md` and follow it to teach that single concept, treating `./<slug>/` as the teaching workspace (work from inside that directory). The concept is the thing to teach; the mission, resources, and learning records are already there for `/teach` to use.
4. **Update progress.** When the lesson is produced, mark the module done (`[x]`) in `CURRICULUM.md` and link the lesson file it produced. Let `/teach` write any learning records.

### 4. Between concepts

After a concept, briefly offer the user three paths: continue to the next module, switch curriculum, or stop for now. Keep `CURRICULUM.md` as the single source of truth for progress so any future session can resume cleanly.

## Managing curricula

- **Switch:** `/study <other topic>` re-points `ACTIVE` (creating the curriculum if new).
- **List:** when there's no clear active curriculum, show the slugs and their progress and let the user choose.
- **Revise:** missions and syllabi change. When the user's goal shifts, update `MISSION.md` and re-sequence `CURRICULUM.md` (confirm structural changes first), and let `/teach` record a learning record for the shift.
