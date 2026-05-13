---
name: implement
description: Run the autonomous implement loop on a Linear parent issue - iterates through its sub-issues, working on each in turn until all are done. Use when the user asks to "implement ABC-123" or "run implement".
argument-hint: "<parent-issue-id> [max-iterations]"
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/specular-ralph:*), Bash(tail:*), Bash(ls:*)
---

Launch the implement loop and schedule a one-time health check.

1. Start the driver in the background via Bash with `run_in_background: true`:

   ```
   "${CLAUDE_PLUGIN_ROOT}/bin/specular-ralph" "${CLAUDE_PLUGIN_ROOT}/skills/implement/specular-ralph.md" $ARGUMENTS
   ```

   **Do NOT also Bash-invoke `bin/specular-ralph` in the foreground.** The driver holds a per-parent file lock; a duplicate invocation will be rejected.

2. Immediately after launching, schedule a one-time reminder 60 seconds from now to check the loop's status. Use natural-language scheduling (e.g. "in 60 seconds, check that the specular-ralph background bash is still progressing"). When the reminder fires, read recent output from the background shell and confirm the headless `claude` process is producing stream-json events. If it appears stuck (no events, repeated permission-prompt lines, or the process has exited with an error), surface that to the user. Otherwise report a brief "still running" and stop.

3. The loop runs to completion in the background; the user sees its streamed output directly. You do not need to babysit it beyond the single 60s check.
