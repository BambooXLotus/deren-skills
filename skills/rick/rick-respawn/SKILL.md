---
name: rick-respawn
description: Boot the current session from a prior /rick-save briefing — read the save, summarize in 3–5 lines, report whether the prior session was in Rick mode. Use at the start of a session that should pick up where the last one left off.
argument-hint: '[optional: folder name (e.g. "48-rotate-share-token" or "auth"), explicit path, or empty for newest across all folders]'
---

# Rick Respawn

Boots the current session from a prior `/rick-save` briefing. Saves live at `docs/rick/<folder>/saves/<timestamp>.md`. Each save is a "Rick" from a parallel timeline; each folder is a feature thread.

## Protocol

### 1. Resolve which Rick

The argument decides what gets loaded.

- **No argument:** find the newest save across all feature folders. Discovery: `find docs/rick -type f -path '*/saves/*.md' 2>/dev/null | sort -r | head -1`. (The trailing path-segment timestamp sort works because filenames are `YYYY-MM-DD_HHMM.md`.)
- **Argument contains `/` or ends with `.md`:** treat it as an explicit path. Use it as-is. (Explicit paths can point anywhere, including archived Ricks outside the working tree.)
- **Argument matches `^[a-z0-9][a-z0-9-]*$`:** treat as a folder name. Find the newest file in `docs/rick/<arg>/saves/`: `ls -1 docs/rick/<arg>/saves/ 2>/dev/null | grep '\.md$' | sort -r | head -1`.

Error cases:

- **No rick directory or no saves anywhere:** print `No Ricks found under docs/rick/. Run /rick-save at the end of a session to create one.` and stop.
- **Folder given, no saves there:** print `No Rick in folder "<folder>". Available folders: <comma-separated list of docs/rick/*/saves/ parents that contain at least one .md>.` and stop.
- **Explicit path does not exist:** print `No save found at <path>.` and stop.

### 2. Read the briefing

Read the entire file. Internalize every section present. Current rick-save shape: `Read first`, `Current state`, `What just happened`, `Open threads`, `Lessons surfaced`, `Suggested skills`, `What NOT to do` (some are skippable).

Parse the folder and timestamp from the path: `docs/rick/<folder>/saves/<YYYY-MM-DD_HHMM>.md`. Folder is the grandparent directory name.

### 3. Detect the Rick trailer

If the file ends with the literal line `You are Rick. You know what to do.` the prior session was in Rick mode. Report this in the summary (Step 4) as `Prior Rick mode: ON` — the trailer is a signal, not an auto-activator. The new session is NOT in Rick mode unless the user fires `/rick-mode` themselves (the skill has `disable-model-invocation: true` and cannot be triggered programmatically by design — persona toggles are user-only).

If the trailer is absent, report `Prior Rick mode: OFF` and behave normally for this session.

### 4. Confirm and wait

Print a 3 to 5 line summary in this exact format:

```
Resumed Rick "<folder>" from <path> (<timestamp>).
Situation: <one line, derived from Current state>
Left off: <one line, the concrete next step — top item from Open threads>
[Critical: <one item from Open threads or What NOT to do>] (only if something is genuinely blocking)
Prior Rick mode: ON | OFF   (fire /rick-mode yourself to continue in that voice)
```

Then stop. Wait for the user's next instruction. Do not start working on the next step until told to.

## Rules

- No em dashes. Periods, commas, parentheses.
- Do not echo the full briefing back. The user wrote it. Reading it back is noise.
- If the briefing is older than 14 days, add `Stale: briefing is N days old, context may have drifted.` before the Rick mode line.
- Surface at most one Critical item. If nothing is genuinely blocking, omit the Critical line entirely.
- Do not commit, push, run tests, or take any side effects on respawn. Just absorb and wait.
- If the briefing references files that no longer exist (deleted, renamed), add a `Drift: <file> not found at expected path` line before Rick mode.
