# study-teach

Two skills for learning anything with Claude Code.

- **`/teach <concept>`** - teach a single concept inside a stateful teaching workspace (mission, trusted resources, beautiful HTML lessons, learning records, reference docs). A near-verbatim copy of [Matt Pocock's `teach` skill](https://github.com/mattpocock/skills/blob/main/skills/productivity/teach/SKILL.md).
- **`/study [topic]`** - run a **curriculum**: an ordered path through many concepts. It picks the next concept and delegates the actual teaching to `/teach`, creating a curriculum on the fly when one doesn't exist and tracking progress across sessions.

## How they relate

`/teach` teaches one thing well. `/study` decides what to learn next and keeps the long arc:

- No active curriculum → `/study` lists what you have and asks which to study.
- No curricula at all → `/study product engineering concepts` builds one and makes it active.
- Resuming → `/study` (no args) picks up the active curriculum where you left off.

State lives in the current directory (just like `/teach`): each curriculum is a `<curriculum>/` subdirectory that is itself a `/teach` workspace plus a `CURRICULUM.md` syllabus that sequences the concepts.

## Credit

The `/teach` skill is the work of [Matt Pocock](https://github.com/mattpocock/skills). `/study` is a thin stateful layer on top of it.
