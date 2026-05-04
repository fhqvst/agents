---
name: implement
description: Run the autonomous implement loop on a Linear parent issue - iterates through its sub-issues, working on each in turn until all are done. Use when the user asks to "implement ABC-123" or "run implement".
argument-hint: "<parent-issue-id> [max-iterations]"
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/specular-ralph:*)
---

> **DO NOT also Bash-invoke `bin/specular-ralph` yourself.** The `!` line below already runs the loop when this skill fires. The loop has a per-parent file lock and a duplicate invocation will be rejected (and is wasted effort either way). To inspect a running loop, read its tee'd stream-json file (each driver `tee`s into a `$TMPDIR/tmp.*` path) instead of starting another driver.

!`"${CLAUDE_PLUGIN_ROOT}/bin/specular-ralph" "${CLAUDE_PLUGIN_ROOT}/skills/implement/specular-ralph.md" $ARGUMENTS`
