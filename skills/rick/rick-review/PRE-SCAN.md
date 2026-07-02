# Shell Pre-Scan patterns (Step 4.5)

Mechanically obvious nits — catchable in shell, no agent needed. Each hit becomes a verified P3 finding appended to the report's P3 table (Step 7).

Run on **added lines only** (`+` prefix in the broad slice) so pre-existing code isn't re-flagged. Citations track the actual source file via `+++ b/<path>` markers and the new-file line number via `@@` hunk headers — not the slice path.

## Patterns

| Pattern matched | Issue text | Fix suggestion |
|---|---|---|
| `console\.(log|error|warn|info|debug)` | console statement in new code | Replace with the project's logger (or delete if debug) |
| `//\s*(TODO|FIXME|XXX|HACK)` | new TODO/FIXME marker | Track in an issue or remove before merge |
| `@ts-(ignore|expect-error)` | TS escape hatch | Hunt the root cause; suppression is debt |
| ` as any\b` | `as any` cast | Tighten the type or use `as unknown as T` if a real type-system boundary |

## Awk script

Run after Step 4 generates `/tmp/rr-diff-broad.txt`. Portable across BSD and GNU awk (no `\b`, no PCRE lookahead, no `\x` escapes):

```bash
PRESCAN=/tmp/rr-prescan.txt
: > $PRESCAN

cat > /tmp/rr-prescan.awk << 'AWKEOF'
BEGIN { line = 0; file = "" }
/^\+\+\+ b\// { file = substr($0, 7); line = 0; next }
/^--- / { next }
/^@@ / {
  if (match($0, /\+[0-9]+/)) {
    line = substr($0, RSTART + 1, RLENGTH - 1) + 0
  }
  next
}
/^\+/ {
  content = substr($0, 2)
  current = line
  line++

  if (match(content, /console\.(log|error|warn|info|debug)/)) {
    printf "| P3 | %s:%d | console statement in new code | Replace with project logger or delete |\n", file, current >> OUT
  }
  if (match(content, /\/\/[[:space:]]*(TODO|FIXME|XXX|HACK)/)) {
    printf "| P3 | %s:%d | new TODO/FIXME marker | Track in an issue or remove before merge |\n", file, current >> OUT
  }
  if (match(content, /@ts-(ignore|expect-error)/)) {
    printf "| P3 | %s:%d | TS escape hatch (@ts-ignore / @ts-expect-error) | Hunt root cause; suppression is debt |\n", file, current >> OUT
  }
  if (match(content, / as any[^a-zA-Z]/)) {
    printf "| P3 | %s:%d | `as any` cast | Tighten the type or use `as unknown as T` |\n", file, current >> OUT
  }
}
/^ / { line++ }
AWKEOF

awk -v OUT="$PRESCAN" -f /tmp/rr-prescan.awk /tmp/rr-diff-broad.txt
```

## Output

If `$PRESCAN` is empty, skip the prepend (per SKILL.md Step 5's "each when non-empty" rule) and the P3-table append is a no-op.

Otherwise wrap the rows with this header so they slot cleanly into Step 7's P3 table:

```
| Severity | File:Line | Issue | Fix |
|----------|-----------|-------|-----|
<contents of $PRESCAN>
```

Append this wrapped block directly to the canonical report's P3 table in Step 7. SKILL.md Step 4.5 defines the separate agent-prepend wrap of `$PRESCAN` held as `$PRESCAN_FINDINGS`.
