# CURRICULUM.md Format

`CURRICULUM.md` lives at the root of a curriculum workspace (`./<slug>/`). It is the **syllabus**: the ordered set of concepts that carry the user from where they are to the mission, plus the progress through them. It is the single source of truth `/study` reads to decide what to teach next, and that any future session reads to resume.

## Template

```md
# Curriculum: {Topic}

Mission: [[MISSION.md]] — {one-line restatement of the goal}

## Modules

1. [x] {Concept} → lessons/0001-slug.html  (2026-06-29)
2. [>] {Concept}
3. [ ] {Concept}
4. [ ] {Concept}
```

Status markers:

- `[ ]` not started
- `[>]` in progress (at most one at a time)
- `[x]` done — append the lesson file it produced and the date

## Rules

- **One concept per module.** A module must be small enough to teach in a single `/teach` lesson. If it needs two lessons, it is two modules.
- **Ordered for the zone of proximal development.** Each module should build on the ones before it. Sequence is the whole point - a bag of unordered topics is not a curriculum.
- **Grounded in the mission.** Every module should trace back to [[MISSION.md]]. Cut modules that don't.
- **Living document.** Re-sequence, add, split, or cut modules as the user's understanding (and the `learning-records/`) evolve. Confirm structural changes with the user.
- **Progress lives only here.** Don't track completion in two places. `CURRICULUM.md` is authoritative; lessons and learning records are the artifacts it points at.
- **Keep it scannable.** The whole syllabus should fit on a screen or two. If it sprawls, the curriculum is too big - consider splitting it into two.
