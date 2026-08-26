---
name: rick-review
description: Multi-agent Rick & Morty council code review for TypeScript projects — up to 7 specialized agents review the diff in parallel, writes a versioned report at docs/rick/<folder>/review/current.md. Use before merging a branch, or call with a PR number to review that PR.
argument-hint:
  [PR number, or leave empty to review current branch / staged changes]
---

# Rick Review

Multi-agent code review. Up to seven Rick & Morty council agents review in parallel — three always run, four are routed by what the diff actually touches. You orchestrate, run three **skepsis** passes (cite → relevance → intent), and write a versioned report.

## Quick Start

```
/rick-review            # reviews current branch / staged changes
/rick-review 213        # reviews PR #213
/rick-review --committed-only  # force committed-only scope even if working tree is dirty
```

## Branches

Four implicit branches. Each short-circuits later steps; read this map first so you don't ambush yourself with a step that no longer applies.

- **Mode:** PR (`$ARGUMENTS=<number>`) vs current-branch. PR mode triggers Step 1's branch auto-swap and Step 3.5(b)'s PR-body extract.
- **Council size:** rickest rick (small, non-sensitive diff) vs full council. Rickest skips Step 5.5 deny-list and collapses Step 6's contradiction check.
- **Version:** N=1 vs N≥2. N=1 skips Step 5.5 (no prior kills to echo).
- **Intel:** with vs without rick-intel dossier (Step 1.5). Affects the Step 8 header line and Step 7.7's intent cross-reference.

## Labels

Canonical label vocabulary — Step 5, Step 8, Step 4.7, and OUTPUT.md all reference these. Don't redefine.

| Label                           | When                                                                         |
| ------------------------------- | ---------------------------------------------------------------------------- |
| `Skipped (out of scope)`        | Agent's trigger didn't fire (e.g. sec slice empty → Evil Morty skipped)      |
| `Skipped (rickest rick)`        | Council slot bypassed because `$RICKEST=true`                                |
| `working-tree`                  | Diff scope: merge-base to working tree (catches uncommitted edits)           |
| `committed-only`                | Diff scope: `$BASE...HEAD`                                                   |
| `committed-only (rickest rick)` | Rickest gate fired on top of committed-only scope (suffix added in Step 4.7) |

## Step 1 — Detect Context

```bash
git branch --show-current
git status --short
gh pr view --json number,title,headRefName,baseRefName 2>/dev/null
```

If `$ARGUMENTS` starts with a number: `gh pr view ${ARGUMENTS%% *} --json number,title,headRefName,baseRefName` (the `%% *` drops trailing tokens like `--committed-only` that Step 4 reads separately; passing the whole `$ARGUMENTS` makes gh error on unknown flags). Capture `baseRefName` as `$BASE`. Stacked PRs diff against their direct parent, not `main`.

**Branch auto-swap (PR mode only):** if `headRefName` from the PR differs from the current local branch, run `gh pr checkout {number}`. This fetches the PR head (handles forks correctly) and switches to it; git refuses the checkout when uncommitted work would be clobbered, so surface that error and stop if it fails. On success, print `Switched to {headRefName} for PR #{number}` and continue. Never silently review the wrong branch.

If no PR: detect default branch with `git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'`. Fallback to `main` if that fails. Use as `$BASE`.

## Step 1.5 — Intel Lookup

Check for an intel dossier per [`../rick-intel/LOOKUP.md`](../rick-intel/LOOKUP.md). If one exists, read its Story Context + Scope. Hold the AB-id, parent epic title, and one-line scope summary for inclusion in the report header (Step 8). The council agents review the _diff_, not the story — don't inject intel into agent prompts. No intel? Skip silently; the report header omits the story line.

## Step 2 — Resolve `REVIEW_NAME`

See [RESOLVE-FOLDER.md](RESOLVE-FOLDER.md). Same folder identity threads through `/rick-plan`, `/rick-save`, `/rick-fix`, `/rick-review-comments`, `/rick-recap`.

## Step 3 — Output Path and Version

Canonical report: `docs/rick/<REVIEW_NAME>/review/current.md` (overwritten each run).

Versioning: list `docs/rick/<REVIEW_NAME>/review/v*.md`, parse the integer from each `v<N>.md` (ignore `-pre` suffixes), `N` = max integer + 1. Sort by parsed integer, not lexically — `v10.md` sorts before `v2.md` and `tail -1` gives `v9.md` at v10+, silently overwriting an existing snapshot. If no prior versions, this is `v1`.

Output files (full list and shape in Step 8 + OUTPUT.md): `current.md`, `VERSIONS.md`, `v<N>.md`, `agents/v<N>/<agent-slug>.md`. Create `docs/rick/<REVIEW_NAME>/review/agents/v<N>/` before launching agents.

## Step 3.5 — Gather System Context

Before launching agents, collect context they can't safely infer. Each subsection is gated — skip when its trigger doesn't fire. Section (a) contributes to `$SYSTEM_CONTEXT` (prepended to every agent's prompt in Step 5). Section (b) is held as `$PR_INTENT` for Step 7.7's cross-reference only.

### a) Settled FIXED findings — only when prior versions exist

```bash
SETTLED_FIXED=$(awk '/✓ FIXED/' docs/rick/<REVIEW_NAME>/review/v*.md 2>/dev/null | sort -u)
```

If non-empty, wrap in this block and hold as `$SYSTEM_CONTEXT`:

```
Settled decisions from prior reviews of this branch (do NOT recommend reverting without new evidence):
<contents of $SETTLED_FIXED>
```

### b) PR description intent — only when a PR was detected in Step 1

PR bodies routinely explain _why_ something looks missing — scope splits ("except seed script"), deferred validation, intentional trade-offs. Read once upfront so Step 7.7 can cross-reference. Do not inject into agent prompts — PR narratives anchor agents on validating stated intent instead of scanning for anomalies the author didn't call out. Same reason Step 1.5 keeps intel out of agent prompts.

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

Hold as `$PR_INTENT` for Step 7.7 only. If no PR was detected, `$PR_INTENT` stays empty and Step 7.7 skips the PR body block.

## Step 4 — Build diff slices

Produce `/tmp/rr-files.txt` plus four per-agent slice files (`broad`, `data`, `api`, `sec`). Full procedure (scope detection, slices, slice cap) in [SLICING.md](SLICING.md).

**Completion:** `/tmp/rr-files.txt` and `/tmp/rr-diff-{broad,data,api,sec}.txt` exist; `$BASE_REF` and `$SCOPE_LABEL` set.

## Step 4.5 — Shell Pre-Scan

Catch mechanically obvious nits before any agent fires. See [PRE-SCAN.md](PRE-SCAN.md) for the pattern catalog (`console.*`, `// TODO/FIXME`, `@ts-ignore`, ` as any`) and the awk script that produces `$PRESCAN`. Hits go two places: appended directly to the P3 table in Step 7, and wrapped in this block held as `$PRESCAN_FINDINGS` (prepended to every agent's prompt in Step 5, so agents don't burn tokens re-searching for what shell already caught):

```
Shell pre-scan already caught these — the orchestrator will file them at P3. Do NOT re-flag:

<contents of $PRESCAN, table rows verbatim>
```

## Step 4.7 — Rickest Rick Mode

The full council is overkill on a 10-line bug fix. **Rickest rick mode** routes small, non-sensitive diffs to one Rick C-137. "Rickest" = the smartest Rick across the multiverse, not the fastest: one top-tier Rick covers every lens better than three generalist reviewers plus specialists.

Run the rickest gate per [SLICING.md](SLICING.md#rickest-gate-step-47). True when: diff ≤ 50 lines AND ≤ 5 files AND specialized slices (data/api/sec) all empty.

When `$RICKEST=true`:

- Step 5 launches only Rick C-137.
- Step 5.5 skipped (single agent, nothing to echo).
- Step 6's cross-agent contradiction check collapses (the cite check still runs).
- Other six council slots render as `Skipped (rickest rick)` per Labels.

**Completion:** `$RICKEST` set; `$SCOPE_LABEL` suffixed if true.

## Step 5 — Route the Council (parallel)

**If `$RICKEST=true` from Step 4.7:** launch only the Rick C-137 row below. Treat every other row as `Skipped (rickest rick)`. Skip Step 5.5. Context assembly still runs — Rick C-137 gets the same substitutes and prepends, just solo.

Read [AGENTS.md](AGENTS.md) for each agent's full prompt template. Two mechanics: **substitute** `{}` placeholders into the template, **prepend** `$` blocks in front of it. All triggered agents get the same context regardless of slice.

**Substitute** (placeholders declared in AGENTS.md):

- `{CHANGED_FILES_LIST}` — full `/tmp/rr-files.txt`
- `{SCOPED_DIFF}` — the agent's slice file from the table below
- `{CLAUDE_MD_CORE}` — `CLAUDE.md` excerpt (read once, inject all agents)
- `{REVIEW_NAME}` — from Step 2
- `{SCOPE_LABEL}` — from Labels

**Prepend** (blocks bolted onto the front of the prompt, each when non-empty):

- `$SYSTEM_CONTEXT` — Step 3.5(a) settled FIXED (revert-prevention)
- `$PRESCAN_FINDINGS` — Step 4.5 shell-verified nits (broad-slice agents must NOT re-scan for these)
- `$DENY_LIST` block — Step 5.5, N≥2 full council only (kill-prevention)

| Agent          | Slice                    | Trigger                                                                                                                                                                                                                |
| -------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rick C-137     | `/tmp/rr-diff-broad.txt` | Always                                                                                                                                                                                                                 |
| Doofus Rick    | `/tmp/rr-diff-broad.txt` | Always                                                                                                                                                                                                                 |
| Smart Morty    | `/tmp/rr-diff-broad.txt` | Always                                                                                                                                                                                                                 |
| Rick Prime     | `/tmp/rr-diff-broad.txt` | `grep -qE '^\+(import\|export).*from .*\.\./' /tmp/rr-diff-broad.txt` OR `comm -13 <(git ls-tree $BASE --name-only src/ \| sort) <(git ls-tree HEAD --name-only src/ \| sort) \| grep -q .` (new top-level `src/` dir) |
| Pickle Rick    | `/tmp/rr-diff-data.txt`  | `[ -s /tmp/rr-diff-data.txt ]`                                                                                                                                                                                         |
| Scientist Rick | `/tmp/rr-diff-api.txt`   | `[ -s /tmp/rr-diff-api.txt ]`                                                                                                                                                                                          |
| Evil Morty     | `/tmp/rr-diff-sec.txt`   | `[ -s /tmp/rr-diff-sec.txt ]` OR `grep -qE 'password\|token\|secret\|jwt\|session\|encrypt' /tmp/rr-diff-broad.txt`                                                                                                    |

**Model.** Omit the `model` field on every Agent call — all agents inherit the parent session's model.

Launch all triggered agents in a single batched Agent tool call with `subagent_type: general-purpose`. Untriggered agents → `Skipped (out of scope)` (per Labels), not `Clean`.

## Step 5.5 — Cross-version Deny-list (N >= 2, full council only)

Skipped when N=1 or `$RICKEST=true`. Otherwise, gather every prior "Killed in Step 7.5" bullet so agents stop re-raising findings the relevance check already discarded. Agents are stateless across versions: on `deren-starter`'s `63-sentry-wiring` branch, v11 through v14 each re-killed the same hedge-gated finding, burning council budget every round.

Kill-prevention lens (don't re-raise parasites). Step 3.5(a) is the revert-prevention lens (don't undo FIXED decisions). Both ship, they catch different misbehaviors.

```bash
DENY_LIST=$(awk '/^### Killed in Step 7\.5/{flag=1; next} /^#+ /{flag=0} flag && /^- /' docs/rick/<REVIEW_NAME>/review/v*.md | sort -u)
```

Empty? Skip the preamble. Otherwise prepend exactly this block to every agent's prompt, before the Step 5 placeholders:

```
Prior reviews of this branch killed the following findings via the Step 7.5 relevance check. Do NOT re-raise them unless the cited code has materially changed since the kill. Hedge-gated kills are particularly likely to look novel on re-read; they are not.

<contents of $DENY_LIST, bullets verbatim>
```

`sort -u` dedupes kills that recurred across multiple prior versions. The awk stop condition `/^#+ /` catches any subsequent heading at any level.

## Step 6 — Skepsis: cite check

Every finding is a **parasite** until proven real. First skepsis pass: does the cite match the code as it stands today?

**Short-circuit:** every agent returned `Clean` → skip to Step 7.

**Pre-existing code filter (mandatory).** For each finding, locate the cited line in the diff slice the agent reviewed:

- Appears as a `+` (added) or `-`/`+` pair (modified) → in scope.
- Appears as context (no prefix) or absent from the slice → existed identically on `$BASE` → **kill**.
- Whitespace-only changes don't count as "changed."

**Verification budget.** Apply the per-severity budget and grep recipe in [PARASITE-CHECK.md](PARASITE-CHECK.md). When `$RICKEST=true`, the budget still applies — only the cross-agent contradiction step (P0 check 4) is vacuous.

**Completion:** every finding marked REAL / PARASITE / DOWNGRADE; parasites removed silently from the report (don't list them).

## Step 7 — Aggregate

1. Dedupe findings only when (a) file matches AND (b) line ranges overlap or are equal (treat `47` and `47-50` as overlapping) AND (c) the Issue text names the same defect. Different defects at the same line stay separate. Mark anything flagged by 2+ agents with `**` — cross-cutting confirmation.
2. Group P0 / P1 / P2 / P3 (CRITICAL / HIGH / MEDIUM / LOW).
3. Within each tier, sort cross-cutting (`**`) rows to the top.
4. Append `$PRESCAN` rows (Step 4.5) directly to the P3 table — shell-verified, no parasite check needed. If a row's `file:line` exactly matches an agent-reported P3 with the same defect, drop the duplicate (keep the shell-verified one — its citation is mechanically guaranteed).

## Step 7.5 — Skepsis: relevance check

Step 6 answered "is the cite real." This step answers "does it matter in production today, before any future change?"

**Auto-keep.** Any row with `**` (cross-cutting) survives. Skip checks.

For each non-`**` row, run the primary check first; auto-kills are mechanical fallbacks.

**Check 1 — Impact restatement (primary).** Restate the finding in one sentence answering "what fails in production today, before any future change?" If the honest restatement needs "would," "could," "if," "after" → **demote one tier**. P3 demoted here is **killed**.

**Auto-kill 1 — Self-defeat scan.** Issue text contains "already covered," "tested elsewhere," "not a real defect," "documentation-level," "structurally cannot" → **KILL**. The agent told you it's not real.

**Auto-kill 2 — Hedge scan.** Issue text contains "if a future edit," "in case someone changes," "for symmetry," "for parity," "for completeness," "hypothetical" → **demote one tier**. P3 demoted is **killed**. Don't add present-tense consequence phrases ("could silently," "after a refactor") to this list — those describe real-today bugs, Check 1 handles them.

Re-number rows after kills and demotions. Tier counts Step 8 prints to chat and writes to VERSIONS.md are these post-7.5 numbers, not Step 7's raw aggregate. Append in the report:

```
### Killed in Step 7.5 (real, but didn't matter)
- [one-line summary]: [which check killed it]
```

(The exact `### Killed in Step 7.5` heading is the anchor Step 5.5's deny-list awk script greps for. Don't rename it.)

**Completion:** every non-`**` row evaluated; kills and demotions applied; sub-heading appended (or omitted when no kills).

## Step 7.7 — Skepsis: intent check

Final skepsis pass: does the surviving finding contradict **documented intent** — the PR description body (`$PR_INTENT` from Step 3.5(b)) or the rick-intel dossier (Step 1.5)?

Skip if neither source exists.

See [INTENT-CROSS-REF.md](INTENT-CROSS-REF.md) for the full demote / strengthen / mitigation / scope-flag rules across both sources.

**Completion:** every P0/P1 survivor compared against `$PR_INTENT` and the intel dossier; row either annotated (`(addressed in PR body)`, `(out of scope per intel)`, `(strengthened — PR body promised X)`) or left intact. Tier counts Step 8 prints are post-7.7.

## Step 8 — Write Report and Print

**Verdict:** `NEEDS FIXES` if any P0 or P1 survives Step 7.5. Otherwise `APPROVED`.

### Files to write (single batch)

All independent — write in one assistant response with parallel tool calls:

- `docs/rick/<REVIEW_NAME>/review/current.md` (canonical)
- `docs/rick/<REVIEW_NAME>/review/v<N>.md` (snapshot of the canonical)
- `docs/rick/<REVIEW_NAME>/review/agents/v<N>/<agent-slug>.md` (one per ran agent)
- Append to `docs/rick/<REVIEW_NAME>/review/VERSIONS.md` via the **Edit tool** — never Write (Write would clobber v1…v(N-1) history if the payload contains only the new line):

```
- v<N> | <YYYY-MM-DD> | <SCOPE_LABEL> | P0:<n> P1:<n> P2:<n> P3:<n> | <APPROVED|NEEDS FIXES>
```

### Canonical report shape

See [OUTPUT.md](OUTPUT.md) for the full structure and VERSIONS.md entry format. The Agents column uses the Labels vocabulary from the top of this skill.

### Print to chat

```
Reviewed <REVIEW_NAME> (v<N>): P0:<n> P1:<n> P2:<n> P3:<n>. Verdict: <APPROVED|NEEDS FIXES>.
Report: docs/rick/<REVIEW_NAME>/review/current.md
```

Then stop. No "want me to start fixing the P0s." That's `rick-fix`'s job.
