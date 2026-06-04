---
name: rick-respawn
description: Boots the current Claude session from a prior /rick-save briefing in docs/rick/saves/. With no argument loads the most recent Rick across all topics. With a slug loads the most recent Rick matching that slug. With a path loads an explicit file. Reads the briefing, summarizes in 3 to 5 lines including the topic, and activates Rick mode if the file ends with the Rick signal. Use when starting a new session that should pick up where a prior session left off, when user invokes /rick-respawn, says "respawn", "resume from save", "pick up from last time", or "load the briefing".
argument-hint: [optional: slug (e.g. "auth"), explicit path, or empty for newest across all topics]
---

# Rick Respawn

Boots the current session from a prior `/rick-save` briefing. Saves live at `docs/rick/saves/<timestamp>_<slug>_rick-save.md`. Each save is a "Rick" from a parallel timeline.

## Protocol

### 1. Resolve which Rick

The argument decides what gets loaded.

- **No argument:** find the newest file under `docs/rick/saves/` by filename sort. That is the most recent Rick across all topics.
- **Argument contains `/` or ends with `.md`:** treat it as an explicit path. Use it as-is. (Explicit paths can point anywhere, including archived Ricks outside the working tree.)
- **Argument matches `^[a-z0-9][a-z0-9-]*$`:** treat as a slug. Find the newest file whose name matches `*_<slug>_rick-save.md`.

Discovery command: `ls -1 docs/rick/saves/ | grep '_rick-save\.md$' | sort -r`. First line is newest.

Error cases:

- **Saves directory missing or empty:** print `No Ricks found in docs/rick/saves/. Run /rick-save at the end of a session to create one.` and stop.
- **Slug given, no match:** print `No Rick tagged "<slug>" found. Available slugs: <comma-separated list from filenames>.` and stop.
- **Explicit path does not exist:** print `No save found at <path>.` and stop.

### 2. Read the briefing

Read the entire file. Internalize every section present. Current rick-save shape: `Read first`, `Current state`, `What just happened`, `Open threads`, `Lessons surfaced`, `Suggested skills`, `What NOT to do` (some are skippable).

Parse the topic and timestamp from the filename:

- Format `<ts>_<slug>_rick-save.md` (4 underscore-delimited parts including the `.md` tail). Slug is the third part.
- If you encounter a different shape, label as `legacy` in the summary and read whatever sections are present.

### 3. Detect the Rick trailer

If the file ends with the literal line `You are Rick. You know what to do.` then Rick mode is active for this session. Invoke `/rick-mode` (also in this plugin) to activate the voice and review protocol. The trailer is just the signal.

If the trailer is absent, behave normally for this session.

### 4. Confirm and wait

Print a 3 to 5 line summary in this exact format:

```
Resumed Rick "<slug>" from <path> (<timestamp>).
Situation: <one line, derived from Current state>
Left off: <one line, the concrete next step — top item from Open threads>
[Critical: <one item from Open threads or What NOT to do>] (only if something is genuinely blocking)
Rick mode: ON | OFF
```

Then stop. Wait for the user's next instruction. Do not start working on the next step until told to.

## Rules

- No em dashes. Periods, commas, parentheses.
- Do not echo the full briefing back. The user wrote it. Reading it back is noise.
- If the briefing is older than 14 days, add `Stale: briefing is N days old, context may have drifted.` before the Rick mode line.
- Surface at most one Critical item. If nothing is genuinely blocking, omit the Critical line entirely.
- Do not commit, push, run tests, or take any side effects on respawn. Just absorb and wait.
- If the briefing references files that no longer exist (deleted, renamed), add a `Drift: <file> not found at expected path` line before Rick mode.

## Stop condition

After printing the summary, stop. Wait for the user.
