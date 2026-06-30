# Slicing — diff scope, per-agent slices, rickest gate

Called from SKILL.md Step 4 (slices) and Step 4.7 (rickest gate).

Run scope detection, slices, and slice cap as **one chained Bash call** — shell vars (`$BASE_REF`, `$BROAD`, etc.) don't persist across separate Bash tool calls, so splitting empties `$BASE_REF` and silently downgrades every later `git diff` to working-tree-vs-HEAD.

## Scope detection

Auto-rule: dirty working tree → `working-tree` scope (catches uncommitted edits); clean → `committed-only`. `--committed-only` in `$ARGUMENTS` forces committed-only regardless.

```bash
if [ -n "$(git status --short)" ] && ! echo "$ARGUMENTS" | grep -q -- '--committed-only'; then
  BASE_REF=$(git merge-base $BASE HEAD)
  SCOPE_LABEL="working-tree"
else
  BASE_REF="$BASE...HEAD"
  SCOPE_LABEL="committed-only"
fi

git diff $BASE_REF --name-only > /tmp/rr-files.txt
```

## Slices

`--diff-filter=d` drops pure-deletion files everywhere.

```bash
BROAD="'*.ts' '*.tsx' ':(exclude)**/*.test.*' ':(exclude)**/*.spec.*' ':(exclude)scripts/**'"
DATA="'**/schema*.ts' '**/migrations/**' '**/models/**' '**/db/**' '**/queries/**' '**/repositor*/**' '*.entity.ts'"
API="'**/routes/**' '**/api/**' '**/server/**' '**/handlers/**' '*.route.ts' '*.handler.ts' '*.controller.ts'"
SEC="'**/auth*/**' '**/middleware*/**' '**/guards/**' '**/security/**' '*.guard.ts' '*.middleware.ts' '.env*'"

eval "git diff $BASE_REF --diff-filter=d -- $BROAD" > /tmp/rr-diff-broad.txt
eval "git diff $BASE_REF --diff-filter=d -- $DATA"  > /tmp/rr-diff-data.txt
eval "git diff $BASE_REF --diff-filter=d -- $API"   > /tmp/rr-diff-api.txt
eval "git diff $BASE_REF --diff-filter=d -- $SEC"   > /tmp/rr-diff-sec.txt
```

## Slice cap (50KB)

Over 50KB → regenerate with `--unified=1`. Still over → hard-truncate with marker so agents know to stop citing past the cut.

```bash
shrink() {
  local f=$1; local pathspec=$2; local LIMIT=51200
  if [ $(wc -c < "$f") -gt $LIMIT ]; then
    eval "git diff $BASE_REF --unified=1 --diff-filter=d -- $pathspec" > "$f"
  fi
  if [ $(wc -c < "$f") -gt $LIMIT ]; then
    head -c $LIMIT "$f" > "$f.tmp"
    printf '\n\n---\n[TRUNCATED at %d bytes. Past this point cite from /tmp/rr-files.txt only; do not flag "missing X".]\n' $LIMIT >> "$f.tmp"
    mv "$f.tmp" "$f"
  fi
}

shrink /tmp/rr-diff-broad.txt "$BROAD"
shrink /tmp/rr-diff-data.txt  "$DATA"
shrink /tmp/rr-diff-api.txt   "$API"
shrink /tmp/rr-diff-sec.txt   "$SEC"
```

## Rickest gate (Step 4.7)

Sets `$RICKEST` and suffixes `$SCOPE_LABEL`. True when the diff is small AND non-sensitive.

```bash
DIFF_LINES=$(git diff $BASE_REF --shortstat | grep -oE '[0-9]+ (insertion|deletion)' | awk '{s+=$1} END {print s+0}')
FILES_CHANGED=$(wc -l < /tmp/rr-files.txt | tr -d ' ')
if [ "$DIFF_LINES" -le 50 ] && [ "$FILES_CHANGED" -le 5 ] \
   && [ ! -s /tmp/rr-diff-data.txt ] \
   && [ ! -s /tmp/rr-diff-api.txt ] \
   && [ ! -s /tmp/rr-diff-sec.txt ]; then
  RICKEST=true
  SCOPE_LABEL="$SCOPE_LABEL (rickest rick)"
else
  RICKEST=false
fi
```
