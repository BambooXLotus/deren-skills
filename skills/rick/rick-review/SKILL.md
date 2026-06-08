---
name: rick-review
description: Multi-agent Rick & Morty council code review for TypeScript projects. Routes between 3 always-run and 4 conditional specialized agents (architecture, patterns, data layer, API contract, security, hygiene, tests) based on the diff scope. Writes a versioned report at docs/rick/<folder>/review/current.md (folder = current branch name, e.g. 48-rotate-share-token), with version snapshots and per-agent raw outputs alongside. Use when user invokes /rick-review, says "review the branch", "council review", "review PR <number>", or wants a structured multi-lens code review before merging.
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

If `$ARGUMENTS` starts with a number: `gh pr view $ARGUMENTS --json number,title,headRefName,baseRefName`. Capture `baseRefName` as `$BASE`. Stacked PRs diff against their direct parent, not `main`.

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

## Step 2 — Resolve `REVIEW_NAME`

This naming must match `rick-fix` Step 1 exactly. Both skills derive it from the same branch state so the read/write round-trip works. It also matches `rick-plan`, `rick-save`, and `rick-recap` so a feature's plan / saves / reviews / recaps all live under the same folder name (e.g. `48-rotate-share-token/`).

Priority order, use the first that produces a value:

1. **Branch name (sanitized).** `git branch --show-current`, replace `/` and spaces with `-`. If non-empty and not in `{main, master, develop}` → `48-rotate-share-token`, `feature-add-payments`. This is the normal path.
2. **PR number.** Only when on main/develop/detached HEAD AND a PR number was passed → `pr-<number>` (e.g. `pr-213`).
3. **Date fallback.** `review-<YYYY-MM-DD>` when nothing else resolves.

Rationale: branch names are stable, human-readable, and symmetric with every other rick-* artifact. Full branch (`48-rotate-share-token`) keeps the slug visible, not just the number, so the folder name still tells you what was in the PR a year later. PR number is still used for branch-verification in Step 1; it's just not the directory name.

## Step 3 — Output Path and Version

Canonical report: `docs/rick/<REVIEW_NAME>/review/current.md` (overwritten each run).

Versioning: list `docs/rick/<REVIEW_NAME>/review/v*.md`. Next `N` = highest + 1. If no prior versions, this is `v1`. Snapshots use the same convention as `rick-improve`: integer `N`, no decimal, no `-loop` suffix.

```
docs/rick/<REVIEW_NAME>/review/
├── current.md                         (canonical, latest)
├── VERSIONS.md                        (append-only history)
├── v<N>.md                            (snapshot of canonical at vN)
└── agents/v<N>/<agent-slug>.md        (per-agent raw output, one per ran agent)
```

Create `docs/rick/<REVIEW_NAME>/review/agents/v<N>/` before launching agents.

## Step 4 — Changed Files and Per-Agent Slices

Auto-rule on scope: if `git status --short` is non-empty AND `--committed-only` was NOT passed, use working-tree scope (catches uncommitted edits). Otherwise committed-only.

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
    printf '\n\n---\n[TRUNCATED at %d bytes — read files directly for the rest.]\n' $LIMIT >> "$f.tmp"
    mv "$f.tmp" "$f"
  fi
}

shrink /tmp/rr-diff-broad.txt "$BROAD"
shrink /tmp/rr-diff-data.txt  "$DATA"
shrink /tmp/rr-diff-api.txt   "$API"
shrink /tmp/rr-diff-sec.txt   "$SEC"
```

If an agent sees the TRUNCATED marker, they cite from `/tmp/rr-files.txt` rather than flagging "missing X."

## Step 5 — Route the Council (parallel)

Read [AGENTS.md](AGENTS.md) for each agent's full prompt template. Inject before launching:

- `{CHANGED_FILES_LIST}` — full `/tmp/rr-files.txt`
- `{SCOPED_DIFF}` — the agent's slice file from the table below
- `{CLAUDE_MD_CORE}` — `CLAUDE.md` excerpt (read once, inject all agents)
- `{REVIEW_NAME}` — from Step 2
- `{SCOPE_LABEL}` — `working-tree` or `committed-only`

| Agent          | Slice                         | Model     | Trigger                                                                                                                       |
| -------------- | ----------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Rick C-137     | `/tmp/rr-diff-broad.txt`      | `haiku`   | Always                                                                                                                        |
| Doofus Rick    | `/tmp/rr-diff-broad.txt`      | `haiku`   | Always                                                                                                                        |
| Smart Morty    | `/tmp/rr-diff-broad.txt`      | `haiku`   | Always                                                                                                                        |
| Rick Prime     | `/tmp/rr-diff-broad.txt`      | inherit   | New top-level dir added under `src/` or cross-module import changes (grep for `from '\\.\\./[^.]` adds in broad slice)        |
| Pickle Rick    | `/tmp/rr-diff-data.txt`       | `haiku`   | Data slice non-empty                                                                                                          |
| Scientist Rick | `/tmp/rr-diff-api.txt`        | `haiku`   | API slice non-empty                                                                                                           |
| Evil Morty     | `/tmp/rr-diff-sec.txt`        | inherit   | Security slice non-empty, or broad slice matches `password\|token\|secret\|jwt\|session\|encrypt`                              |

**Model column.** Pass `model: "haiku"` on the Agent tool call for agents marked `haiku` (Haiku 4.5 — fast, sufficient for pattern-matching review against documented conventions). Omit the `model` field for agents marked `inherit` so they use the parent session's default (typically Sonnet/Opus); Rick Prime (cross-module architectural impact) and Evil Morty (security exploit reasoning) are the two lenses where reasoning depth materially changes verdicts, so they stay on the stronger model. Net effect: ~5x faster council step on routine PRs (5 of 7 agents on Haiku), with no loss of depth where it matters.

Launch all triggered agents in a single batched Agent tool call with `subagent_type: general-purpose`. Agents not triggered are reported as `Skipped (out of scope)` — not `Clean`.

## Step 6 — Total Rickall (parasite check)

Every finding is a parasite until proven real. Verify before aggregating.

### Short-circuit

If every agent that ran returned `Clean`, skip to Step 7.

### Pre-existing code filter (mandatory)

For each finding: is the cited code in the diff slice the agent reviewed? If the line existed identically on `$BASE` and was not added or modified by this PR, it's **out of scope** — kill it. Whitespace-only changes don't count as "changed."

### Verification budget

| Severity | Allowed verification                                                                          |
| -------- | --------------------------------------------------------------------------------------------- |
| P0       | Batched grep, plus deep file read if grep matched. Only tier that gets a full read.           |
| P1       | Batched grep only. If grep doesn't match the citation → kill. No file read.                   |
| P2       | Citation existence check from `/tmp/rr-files.txt`. Kill if file isn't in the diff.            |
| P3       | No verification. Trust the agent or kill on suspicion of hallucination.                       |

Run a single batched grep covering every P0 and P1 citation:

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
- **DOWNGRADE** — real issue, wrong severity. Adjust.

Remove parasites from the report silently. Don't list them.

## Step 7 — Aggregate

1. Dedupe findings at the same `file:line`. Mark anything flagged by 2+ agents with `**` — cross-cutting confirmation.
2. Group P0 / P1 / P2 / P3 (CRITICAL / HIGH / MEDIUM / LOW).
3. Within each tier, sort cross-cutting (`**`) rows to the top.

## Step 7.5 — Final Skeptic Pass

Step 6 answers "does the cite match the code." This step answers "does the finding actually matter today."

### Auto-keep

Any row with `**` survives. Skip checks.

### Three cheap checks (each non-`**` row, in order)

**1. Self-defeat scan.** If the row's Issue text contains "already covered," "tested elsewhere," "not a real defect," "documentation-level," "structurally cannot" — **KILL**. The agent told you it's not real.

**2. Hedge scan.** If the row contains "could silently," "if a future edit," "in case someone changes," "after a refactor," "for symmetry," "for parity," "for completeness," "hypothetical" — **demote one tier**. P3 hits get killed unless prior reviews accepted the same pattern.

**3. Impact restatement.** Restate the finding in one sentence answering "what fails in production today, before any future change?" If the honest restatement needs "would," "could," "if," "after" — **demote one tier**. P3 demoted from this gate is **killed**.

Re-number rows after kills and demotions. Append a sub-heading in Total Rickall:

```
### Killed in Step 7.5 (real, but didn't matter)
- [one-line summary]: [which check killed it]
```

## Step 8 — Write Report and Print

### Files to write (single batch)

All independent — write in one assistant response with parallel tool calls:

- `docs/rick/<REVIEW_NAME>/review/current.md` (canonical)
- `docs/rick/<REVIEW_NAME>/review/v<N>.md` (snapshot of the canonical)
- `docs/rick/<REVIEW_NAME>/review/agents/v<N>/<agent-slug>.md` (one per ran agent)
- Append to `docs/rick/<REVIEW_NAME>/review/VERSIONS.md`:
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
