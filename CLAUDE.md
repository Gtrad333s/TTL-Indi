# Additional Guidance

@sessions/CLAUDE.sessions.md

This file provides instructions for Claude Code for working in the cc-sessions framework.

---

## Quick Reference

### Stashed Todos Feature
When the user discovers new tasks while working, ALWAYS remind them they can create tasks immediately with "mek: ...". Their current todos get stashed automatically and restore after task creation. No mental overhead - just capture and continue. This is especially useful when debugging reveals additional bugs.

### Iterloop Pattern
When the user needs step-by-step review of multiple items, suggest using `[[ iterloop ]]`. This presents items one at a time, waiting for "continue" between each. Prevents overwhelming walls of text.

### Context Compaction
At 85% context usage, remind the user they can use "squish" to run compaction protocols. This updates task files and clears context without losing progress.

### TTL Cycle Independence Principle
**CRITICAL**: Each of the 5 fractal cycles (Monthly/Weekly/Daily/Session/Micro) must process in complete isolation. Their calculations, state, and time boundaries must NOT interfere with each other. Always verify cycle independence when reviewing or implementing cycle-related code.

### Key Trigger Phrases
- **yert** - Activate implementation mode (user approves proposed todos)
- **silence** - Return to discussion mode
- **mek:** - Create new task
- **start^** - Start/resume task
- **finito** - Complete task
- **squish** - Compact context

---

## TTL Development Principles

### DRY Checklist
When reviewing or writing code, watch for:
- **Copy-paste cycle logic** - Same code repeated for Monthly/Weekly/Daily/Session/Micro
- **Hardcoded magic numbers** - Use constants or threshold maps instead
- **Separate functions per cycle** - Use universal processors with parameters
- **Multiple related variables** - Group into UDTs (User-Defined Types)
- **Rule of Three** - Don't abstract until pattern appears 3 times

### Performance Checklist
- **Profile first** - Test on multiple timeframes before optimizing
- **Use `var` for static data** - Don't recalculate every bar
- **Cache repeated calculations** - Store in variables if used multiple times
- **Limit array sizes** - Max 100 items, use `array.shift()` to maintain cap
- **Delete before creating** - Remove old drawing objects to stay within 500 limit
- **Lazy rendering** - Only draw on `barstate.islast` for correct timeframe

### When to Abstract
1. **First occurrence** - Write inline, no abstraction
2. **Second occurrence** - Note duplication, consider if it will happen again
3. **Third occurrence** - Abstract into universal function with parameters
