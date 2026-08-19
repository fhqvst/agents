---
name: work-on-issue
description: Test-driven development with red-green-refactor loop. Invoked by the implement loop to work on a sub-issue, not by users directly.
user-invocable: false
---

# Test-Driven Development

## Philosophy

**Core principle**: Tests should verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't.

**Good tests** are integration-style: they exercise real code paths through public APIs. They describe _what_ the system does, not _how_ it does it. A good test reads like a specification - "user can checkout with valid cart" tells you exactly what capability exists. These tests survive refactors because they don't care about internal structure.

**Bad tests** are coupled to implementation. They mock internal collaborators, test private methods, or verify through external means (like querying a database directly instead of using the interface). The warning sign: your test breaks when you refactor, but behavior hasn't changed. If you rename an internal function and tests fail, those tests were testing implementation, not behavior.

See [tests.md](tests.md) for examples and [mocking.md](mocking.md) for mocking guidelines.

## Anti-Pattern: Horizontal Slices

**DO NOT write all tests first, then all implementation.** This is "horizontal slicing" - treating RED as "write all tests" and GREEN as "write all code."

This produces **crap tests**:

- Tests written in bulk test _imagined_ behavior, not _actual_ behavior
- You end up testing the _shape_ of things (data structures, function signatures) rather than user-facing behavior
- Tests become insensitive to real changes - they pass when behavior breaks, fail when behavior is fine
- You outrun your headlights, committing to test structure before understanding the implementation

**Correct approach**: Vertical slices via tracer bullets. One test → one implementation → repeat. Each test responds to what you learned from the previous cycle. Because you just wrote the code, you know exactly what behavior matters and how to verify it.

```
WRONG (horizontal):
  RED:   test1, test2, test3, test4, test5
  GREEN: impl1, impl2, impl3, impl4, impl5

RIGHT (vertical):
  RED→GREEN: test1→impl1
  RED→GREEN: test2→impl2
  RED→GREEN: test3→impl3
  ...
```

## Workflow

### 1. Planning

This skill runs inside a subagent - there is no user to ask. The spec is already fixed. The implement loop passes in the parent's full body in two pieces: a **human-facing top** (Problem, Proposal, Constraints In/Out, headline Implementation pseudocode) and a `PLAN.md` (user stories, deeper implementation decisions, testing decisions). Both pieces are the brief:

- **Problem and proposal** come from the human-top.
- **Out-of-scope guardrails** come from the human-top's Constraints "Out".
- **Interface changes** come from the human-top's Implementation pseudocode plus `PLAN.md` → "Implementation Decisions" (deeper detail).
- **Which behaviors to test** come from `PLAN.md` → "Testing Decisions" plus the sub-issue's acceptance criteria.
- **Scope** is the single vertical slice described in the sub-issue body. Don't widen it.

Respect ADRs in the area you're touching.

**Read the repo's coding standards before writing code.** Locate them from the repo's `CLAUDE.md` / `AGENTS.md` — they commonly live in a `standards/`, `docs/`, or `.github/` tree, often with a review checklist alongside — and read the ones covering the area you're about to touch (style, testing, and the language-specific cluster). These encode rules that are not lint-enforced and not derivable from the surrounding code, so reading a neighbouring file is not a substitute. Running unattended is not an excuse to skip this: nobody will catch the violation for you.

Never write scratch or temp files into the working tree (no `/tmp/old.ts` stashes, no `*.bak` copies). To recall prior code, use `git show HEAD:path/to/file`. Stray files get swept into commits and leak across iterations.

Before writing any code:

- [ ] Read the repo's coding standards covering the area you're touching
- [ ] Identify opportunities for [deep modules](deep-modules.md) (small interface, deep implementation)
- [ ] Design interfaces for [testability](interface-design.md)
- [ ] List the behaviors to test (not implementation steps), prioritizing the sub-issue's acceptance criteria

**You can't test everything.** Focus testing effort on the acceptance criteria and the critical paths called out in `PLAN.md`'s Testing Decisions, not every possible edge case.

### 2. Tracer Bullet

Write ONE test that confirms ONE thing about the system:

```
RED:   Write test for first behavior → test fails
GREEN: Write minimal code to pass → test passes
```

This is your tracer bullet - proves the path works end-to-end.

### 3. Incremental Loop

For each remaining behavior:

```
RED:   Write next test → fails
GREEN: Minimal code to pass → passes
```

Rules:

- One test at a time
- Only enough code to pass current test
- Don't anticipate future tests
- Keep tests focused on observable behavior

### 4. Refactor

After all tests pass, look for [refactor candidates](refactoring.md):

- [ ] Extract duplication
- [ ] Deepen modules (move complexity behind simple interfaces)
- [ ] Apply SOLID principles where natural
- [ ] Consider what new code reveals about existing code
- [ ] Run tests after each refactor step

**Never refactor while RED.** Get to GREEN first.

## Checklist Per Cycle

```
[ ] Test describes behavior, not implementation
[ ] Test uses public interface only
[ ] Test would survive internal refactor
[ ] Test asserts a concrete, independently-known result (does NOT restate the implementation)
[ ] Code is minimal for this test
[ ] No speculative features added
[ ] Comments describe the code, not the change being made
```

## Comments

You are working an issue, so every comment you write is at risk of describing *the change* rather than *the code*. The issue, the plan, and the diff are invisible to whoever opens this file next month.

Before committing, re-read every comment you added and delete or rewrite any that:

- **Narrate the diff** - `// now uses X`, `// previously we did Y`, `// extracted from Z so that...`, `// this fixes...`
- **Point at transient context** - a PR, ticket id, issue, or the conversation that produced the change
- **Point at unlanded work** - `// safe until sharding ships`, `// a later slice replaces this`. State the constraint as it holds today
- **Repeat a rationale already stated elsewhere** - say it once, on the helper or type it describes, and let each call site carry only its own specific fact

A comment earns its place by explaining something the code cannot say: a hidden constraint, a subtle invariant, an upstream bug being worked around, why an obvious simpler approach fails. Default to no comment.

If the repo has its own comment or style standard, it wins over this section.

**Do not add tests which simply restate the implementation.** A test that computes its expected value with the same formula the code uses, or asserts a constant equals the same constant the code returns, provides zero confidence - it passes by construction. If the only way a test could fail is a typo in the test itself, delete it. See [tests.md](tests.md) for examples.
