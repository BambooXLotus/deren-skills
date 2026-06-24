# VERSIONS.md format (rick-improve)

The append-only history file at `<target-dir>/versions/VERSIONS.md`. Two writes per run: a pre-loop stub before round 1, then a full entry after the loop finishes.

## First-time creation

Start the file with:

```markdown
# Versions: <skill-name>

Append-only history of /rick-improve runs on this skill. Each entry is one bounded loop.
```

## Pre-loop stub (written before round 1)

```
- v<N> | <YYYY-MM-DD> | pre-loop snapshot
```

## Post-loop full entry (appended after the loop finishes)

```
- v<N> | <YYYY-MM-DD> | <stop-condition: 3-rounds | zero-changes | over-cap | needs-rewrite>
  - Round 1: <one line, what changed>
  - Round 2: <one line, or "stopped early">
  - Round 3: <one line, or "stopped early">
  - Restore pre when: <condition that would make v<N>-pre better than v<N>>
  - Restore post when: <condition that would make v<N> the new baseline>
```

## Restore condition shape (binding)

Restore conditions must name a behavior that would be observably different, not "if this is bad." Examples of good restore conditions:

- `Restore pre when: the new structured-finding format makes rounds 2-3 too rigid and we want freeform critique back`
- `Restore post when: zero-changes rounds stop happening — confirms the new "trivial polish doesn't count" rule landed correctly`
