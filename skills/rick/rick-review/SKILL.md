---
name: rick-review
description: Multi-agent Rick & Morty council code review for TypeScript projects — up to 7 specialized agents review the diff in parallel, writes a versioned report at docs/rick/<folder>/review/current.md. Use before merging a branch, or call with a PR number to review that PR.
argument-hint: [PR number, or leave empty to review current branch / staged changes]
---

# Rick Review

Multi-agent code review. Up to seven Rick & Morty council agents review in parallel — three always run, four are routed by what the diff actually touches. You orchestrate, aggregate, parasite-check, and write a versioned report.

## Quick Start

```
/rick-review            # reviews current branch / staged changes
/rick-review 213        # reviews PR #213
/rick-review --committed-only  # force committed-only scope even if working tree is dirty
```

## Step 1 — Detect Context

```bash
git branch --show-current
git status --short
gh pr view --json number,title,headRefName,baseRefName 2>/dev/null
```

If `$ARGUMENTS` starts with a number: `gh pr view ${ARGUMENTS%% *} --json number,title,headRefName,baseRefName` (the `%% *` drops trailing tokens like `--committed-only` that Step 4 reads separately; passing the whole `$ARGUMENTS` makes gh error on unknown flags). Capture `baseRefName` as `$BASE`. Stacked PRs diff against their direct parent, not `main`.

**Branch verification (PR mode only):** compare `headRefName` from the PR against the current local branch. If they differ, stop and tell the user:

```
Branch mismatch.
PR #{number} is on branch: {headRefName}
You are currently on:      {current branch}

Switch first:
  git checkout {headRefName}
```

Never silently review the wrong branch.

If no PR: detect default branch with `git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'`. Fallback to `main` if that fails. Use as `$BASE`.

## Step 1.5 — Intel Lookup

Check for an intel dossier per [`../rick-intel/LOOKUP.md`](../rick-intel/LOOKUP.md). If one exists, read its Story Context + Scope. Hold the AB-id, parent epic title, and one-line scope summary for inclusion in the report header (Step 8). The council agents review the *diff*, not the story — don't inject intel into agent prompts. No intel? Skip silently; the report header omits the story line.

## Step 2 — Resolve `REVIEW_NAME`

See [RESOLVE-FOLDER.md](RESOLVE-FOLDER.md). Same folder identity threads through `/rick-plan`, `/rick-save`, `/rick-fix`, `/rick-review-comments`, `/rick-recap`.

## Step 3 — Output Path and Version

Canonical report: `docs/rick/<REVIEW_NAME>/review/current.md` (overwritten each run).

Versioning: list `docs/rick/<REVIEW_NAME>/review/v*.md`, parse the integer from each `v<N>.md` (ignore `-pre` suffixes), `N` = max integer + 1. Sort by parsed integer, not lexically — `v10.md` sorts before `v2.md` and `tail -1` gives `v9.md` at v10+, silently overwriting an existing snapshot. If no prior versions, this is `v1`. Snapshots use the same convention as `rick-improve`: integer `N`, no decimal, no `-loop` suffix.

Output files (full list and shape in Step 8 + OUTPUT.md): `current.md`, `VERSIONS.md`, `v<N>.md`, `agents/v<N>/<agent-slug>.md`. Create `docs/rick/<REVIEW_NAME>/review/agents/v<N>/` before launching agents.

## Step 3.5 — Gather System Context

Before launching agents, collect context they can't safely infer. Each block is gated — skip when its trigger doesn't fire. Hold the bundle as `{SYSTEM_CONTEXT}` for Step 5 agent prompts, and `$PR_INTENT` separately for Step 7.7.

### a) Settled FIXED findings — only when prior versions exist

```bash
SETTLED_FIXED=$(awk '/✓ FIXED/' docs/rick/<REVIEW_NAME>/review/v*.md 2>/dev/null | sort -u)
```

If non-empty, include in `{SYSTEM_CONTEXT}`:

```
Settled decisions from prior reviews of this branch (do NOT recommend reverting without new evidence):
<contents of $SETTLED_FIXED>
```

### b) PR description intent — only when a PR was detected in Step 1

PR bodies routinely explain *why* something looks missing — scope splits ("except seed script"), deferred validation, intentional trade-offs. Read once upfront so agents can flag contradictions and Step 7.7 can cross-reference.

```bash
PR_BODY=$(gh pr view --json body --jq '.body' 2>/dev/null)
```

If non-empty, extract verbatim text (no rewriting) from these sections:

- `## Summary`
- `## Security Notes` / `## Security Considerations`
- `## Performance Notes` / `## Performance Impact`
- `## Testing Instructions` / `## Testing`
- `## Deployment Notes` / `## Rollback Plan`
- `## Additional Context`
- Any header line matching `WORK ITEM:`

Hold as `$PR_INTENT`. Inject as `{PR_INTENT}` in Step 5. If no PR was detected, both stay empty and Step 7.7 skips the PR body block.

## Step 4 — Changed Files and Per-Agent Slices

Auto-rule on scope: if `git status --short` is non-empty AND `--committed-only` was NOT passed, use working-tree scope (catches uncommitted edits). Otherwise committed-only.

Run Step 4's three bash blocks (scope detection, slices, slice cap) as one chained Bash call — shell vars (`$BASE_REF`, `$BROAD`, etc.) set in one block don't persist across separate Bash tool calls, so splitting them empties `$BASE_REF` and silently downgrades every later `git diff` to working-tree-vs-HEAD.

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

Record `$SCOPE_LABEL` in the report header.

Slices (`--diff-filter=d` everywhere drops pure-deletion files):

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

### Slice cap (50KB)

Cap each slice at 50KB. If over, regenerate with `--unified=1`. If still over, hard-truncate and append a marker:

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

## Step 4.5 — Shell Pre-Scan

Catch mechanically obvious nits before any agent fires. See [PRE-SCAN.md](PRE-SCAN.md) for the pattern catalog (`console.*`, `// TODO/FIXME`, `@ts-ignore`, ` as any`) and the awk script that produces them. Hits are appended directly to the P3 table in Step 7 and injected as `{PRESCAN_FINDINGS}` so the broad-slice agents know to skip these checks.

## Step 5 — Route the Council (parallel)

Read [AGENTS.md](AGENTS.md) for each agent's full prompt template. Inject before launching:

- `{CHANGED_FILES_LIST}` — full `/tmp/rr-files.txt`
- `{SCOPED_DIFF}` — the agent's slice file from the table below
- `{CLAUDE_MD_CORE}` — `CLAUDE.md` excerpt (read once, inject all agents)
- `{SYSTEM_CONTEXT}` — Step 3.5 bundle (settled FIXED + anything else assembled)
- `{PR_INTENT}` — Step 3.5(b) PR body extract (empty when no PR)
- `{PRESCAN_FINDINGS}` — Step 4.5 shell-verified nits (broad-slice agents must NOT re-scan for these)
- `{REVIEW_NAME}` — from Step 2
- `{SCOPE_LABEL}` — `working-tree` or `committed-only`

| Agent          | Slice                         | Model     | Trigger                                                                                                                       |
| -------------- | ----------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Rick C-137     | `/tmp/rr-diff-broad.txt`      | `haiku`   | Always                                                                                                                        |
| Doofus Rick    | `/tmp/rr-diff-broad.txt`      | `haiku`   | Always                                                                                                                        |
| Smart Morty    | `/tmp/rr-diff-broad.txt`      | `haiku`   | Always                                                                                                                        |
| Rick Prime     | `/tmp/rr-diff-broad.txt`      | inherit   | `grep -qE '^\+(import\|export).*from .*\.\./' /tmp/rr-diff-broad.txt` OR `comm -13 <(git ls-tree $BASE --name-only src/ \| sort) <(git ls-tree HEAD --name-only src/ \| sort) \| grep -q .` (new top-level `src/` dir) |
| Pickle Rick    | `/tmp/rr-diff-data.txt`       | `haiku`   | `[ -s /tmp/rr-diff-data.txt ]`                                                                                                |
| Scientist Rick | `/tmp/rr-diff-api.txt`        | `haiku`   | `[ -s /tmp/rr-diff-api.txt ]`                                                                                                 |
| Evil Morty     | `/tmp/rr-diff-sec.txt`        | inherit   | `[ -s /tmp/rr-diff-sec.txt ]` OR `grep -qE 'password\|token\|secret\|jwt\|session\|encrypt' /tmp/rr-diff-broad.txt`            |

**Model column.** Pass `model: "haiku"` on the Agent tool call for agents marked `haiku` (Haiku 4.5 — fast, sufficient for pattern-matching review against documented conventions). Omit the `model` field for agents marked `inherit` so they use the parent session's default (typically Sonnet/Opus); Rick Prime (cross-module architectural impact) and Evil Morty (security exploit reasoning) are the two lenses where reasoning depth materially changes verdicts, so they stay on the stronger model. Net effect: ~5x faster council step on routine PRs (5 of 7 agents on Haiku), with no loss of depth where it matters.

For N >= 2: build the cross-version deny-list per Step 5.5 below and prepend its preamble to every agent's prompt template before the batch call. Skipped when N=1.

Launch all triggered agents in a single batched Agent tool call with `subagent_type: general-purpose`. Agents not triggered are reported as `Skipped (out of scope)` — not `Clean`.

## Step 5.5 — Cross-version Deny-list (N >= 2 only)

Skipped when N=1. For N >= 2, gather every prior "Killed in Step 7.5" bullet so agents stop re-raising findings the skeptic pass already discarded. Agents are stateless across versions: on `deren-starter`'s `63-sentry-wiring` branch, v11 through v14 each re-killed the same hedge-gated "participantName not asserted in extra-scrub test" finding, burning council budget every round.

This is the kill-prevention lens (don't re-raise parasites). Step 3.5(a) is the revert-prevention lens (don't undo FIXED decisions). Both ship, they catch different misbehaviors.

```bash
DENY_LIST=$(awk '/^### Killed in Step 7\.5/{flag=1; next} /^#+ /{flag=0} flag && /^- /' docs/rick/<REVIEW_NAME>/review/v*.md | sort -u)
```

If `$DENY_LIST` is empty (no prior kills), skip the preamble — there's nothing to suppress. Otherwise prepend exactly this block to every agent's prompt, before the placeholders Step 5 injects:

```
Prior reviews of this branch killed the following findings via the Step 7.5 Final Skeptic Pass. Do NOT re-raise them unless the cited code has materially changed since the kill. Hedge-gated kills are particularly likely to look novel on re-read; they are not.

<contents of $DENY_LIST, bullets verbatim>
```

The `sort -u` dedupes kills that recurred across multiple prior versions (the same hedge-gated finding can land four times in a row otherwise). The awk stop condition `/^#+ /` catches any subsequent heading at any level, so the slurp ends at the next `## ` or `### ` boundary.

## Step 6 — Total Rickall (parasite check)

Every finding is a parasite until proven real. Verify before aggregating.

### Short-circuit

If every agent that ran returned `Clean`, skip to Step 7.

### Pre-existing code filter (mandatory)

For each finding: locate the cited line in the diff slice the agent reviewed. If it appears as a `+` line (added) or `-`/`+` pair (modified) by this PR, it's in scope. If it appears as a context line (no `+`/`-` prefix) or isn't in the slice at all, it existed identically on `$BASE` and is **out of scope** — kill it. Whitespace-only changes don't count as "changed."

### Verification budget (applied to prefix-filter survivors only)

| Severity | Allowed verification                                                                          |
| -------- | --------------------------------------------------------------------------------------------- |
| P0       | Batched grep, plus deep file read if grep matched. Only tier that gets a full read.           |
| P1       | Batched grep only. If grep doesn't match the citation → kill. No file read.                   |
| P2       | Citation existence check from `/tmp/rr-files.txt`. Kill if file isn't in the diff.            |
| P3       | No grep, no read. Kill if cited file isn't in `/tmp/rr-files.txt`. Otherwise pass to Step 7.5. |

Run a single batched grep covering every P0 and P1 citation (one pattern per finding, derived from the cited code snippet or named symbol — not a loose keyword like the function name alone):

```bash
grep -nE '<pat1>|<pat2>|<pat3>' <file1> <file2> <file3>
```

For P0 survivors only, read the actual file and confirm:

1. The cited code does what the finding claims.
2. Existing validation / guards / catch blocks don't already handle it.
3. Severity matches blast radius. Downgrade if not.
4. Contradictions between agents — read the code, kill the parasite.

Mark each finding:

- **REAL** — verified, stays
- **PARASITE** — claim doesn't match code, or pre-existing, or already mitigated. Remove.
- **DOWNGRADE** — real issue, wrong severity. Adjust. P0 only — lower tiers don't read code, so severity stays.

Remove parasites from the report silently. Don't list them.

## Step 7 — Aggregate

1. Dedupe findings only when (a) file matches AND (b) line ranges overlap or are equal (treat `47` and `47-50` as overlapping) AND (c) the Issue text names the same defect (e.g. both "missing await", not "missing await" + "unsafe cast"). Different defects at the same line stay separate. Mark anything flagged by 2+ agents with `**` — cross-cutting confirmation.
2. Group P0 / P1 / P2 / P3 (CRITICAL / HIGH / MEDIUM / LOW).
3. Within each tier, sort cross-cutting (`**`) rows to the top.
4. Append `$PRESCAN` rows (from Step 4.5) directly to the P3 table — shell-verified, no parasite check needed. If a row's `file:line` exactly matches an agent-reported P3 with the same defect, drop the duplicate (keep the shell-verified one — its citation is mechanically guaranteed).

## Step 7.5 — Final Skeptic Pass

Step 6 answers "does the cite match the code." This step answers "does the finding actually matter today."

### Auto-keep

Any row with `**` survives. Skip checks.

### Three cheap checks (each non-`**` row, in order)

**1. Self-defeat scan.** If the row's Issue text contains "already covered," "tested elsewhere," "not a real defect," "documentation-level," "structurally cannot" — **KILL**. The agent told you it's not real.

**2. Hedge scan.** If the row contains "if a future edit," "in case someone changes," "for symmetry," "for parity," "for completeness," "hypothetical" — **demote one tier**. P3 demoted from this gate is **killed**. Don't add present-tense consequence phrases ("could silently," "after a refactor") to this list — those describe real-today bugs in plain prose and demoting them flips the Step 8 verdict; Check 3 catches genuinely prospective findings by restatement instead.

**3. Impact restatement.** Restate the finding in one sentence answering "what fails in production today, before any future change?" If the honest restatement needs "would," "could," "if," "after" — **demote one tier**. P3 demoted from this gate is **killed**.

Re-number rows after kills and demotions. The tier counts Step 8 prints to chat and writes to VERSIONS.md are these post-Step 7.5 numbers (after kills and demotions), not Step 7's raw aggregate. Append a sub-heading in Total Rickall:

```
### Killed in Step 7.5 (real, but didn't matter)
- [one-line summary]: [which check killed it]
```

## Step 7.7 — Documented Intent Cross-Reference

Do one final pass over the surviving findings against **documented intent** — the PR description body (`$PR_INTENT` from Step 3.5(b)) and the rick-intel dossier (from Step 1.5). See [INTENT-CROSS-REF.md](INTENT-CROSS-REF.md) for the full demote / strengthen / mitigation / scope-flag rules across both sources.

Skip the whole step if neither `$PR_INTENT` nor an intel dossier exists — no documented intent to cross-reference against. Tier counts Step 8 prints are post-7.7 (after any demotions / strengthenings / removals here).

## Step 8 — Write Report and Print

**Verdict:** `NEEDS FIXES` if any P0 or P1 survives Step 7.5. Otherwise `APPROVED`.

### Files to write (single batch)

All independent — write in one assistant response with parallel tool calls:

- `docs/rick/<REVIEW_NAME>/review/current.md` (canonical)
- `docs/rick/<REVIEW_NAME>/review/v<N>.md` (snapshot of the canonical)
- `docs/rick/<REVIEW_NAME>/review/agents/v<N>/<agent-slug>.md` (one per ran agent)
- Append to `docs/rick/<REVIEW_NAME>/review/VERSIONS.md` via the Edit tool — do NOT use Write (Write would clobber prior v1…v(N-1) review history if the payload contains only the new line):
  ```
  - v<N> | <YYYY-MM-DD> | <SCOPE_LABEL> | P0:<n> P1:<n> P2:<n> P3:<n> | <APPROVED|NEEDS FIXES>
  ```

### Canonical report shape

See [OUTPUT.md](OUTPUT.md) for the full structure and VERSIONS.md entry format.

### Print to chat

```
Reviewed <REVIEW_NAME> (v<N>): P0:<n> P1:<n> P2:<n> P3:<n>. Verdict: <APPROVED|NEEDS FIXES>.
Report: docs/rick/<REVIEW_NAME>/review/current.md
```

Then stop. No "want me to start fixing the P0s." That's `rick-fix`'s job.
