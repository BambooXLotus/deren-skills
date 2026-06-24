---
name: rick-review-comments
description: Reformat the latest /rick-review report into an inline-comments markdown file for the human PR reviewer.
argument-hint: "[<REVIEW_NAME>] — folder under docs/rick/; auto-resolves from current branch if omitted"
disable-model-invocation: true
---

# rick-review-comments

Reads `docs/rick/<REVIEW_NAME>/review/current.md` (the canonical /rick-review output) and writes a sibling `docs/rick/<REVIEW_NAME>/review/current-comments.md` formatted for the human PR reviewer to drop on matching lines.

## Quick start

```
/rick-review-comments                            # auto-resolve REVIEW_NAME from current branch
/rick-review-comments 48-rotate-share-token      # explicit folder
```

## Step 1 — Resolve `REVIEW_NAME`

If `$ARGUMENTS` first token is non-empty, use it. Otherwise resolve per [`../rick-review/RESOLVE-FOLDER.md`](../rick-review/RESOLVE-FOLDER.md).

Verify `docs/rick/<REVIEW_NAME>/review/current.md` exists. If missing:

```
No canonical review at docs/rick/<REVIEW_NAME>/review/current.md. Run /rick-review first.
```

## Step 2 — Gather Context

- Read the canonical review's header (`**Version:** v<N>`, branch, base, scope label, verdict).
- `gh pr view --json number,title,headRefName,baseRefName,body 2>/dev/null` from the current branch. If `headRefName` matches the review's branch, capture PR `number` for the header. If no PR detected, write the header without `#N`.
- Parse the canonical review's tables:
  - **P0 / P1 / P2 / P3 findings** — each row → finding (`#`, agents, `file:line`, issue, fix).
  - **Total Rickall — Killed in Step 7.5** → kill-list items.
- **Carry-overs.** If `docs/rick/<REVIEW_NAME>/review/v<N-1>.md` exists, scan it for finding rows whose `file:line` does NOT appear in `current.md` at all (the diff removed the offending code between versions). Those are carry-overs the reviewer can mark "verified clean." Strikethrough/FIXED rows in `current.md` are classified as "this round" by Step 3 item 4 — do NOT also list them as carry-overs (duplicates the entry in the reviewer's file). If no `v<N-1>.md` exists, carry-overs is 0.

## Step 2.5 — Classify Each Finding by rick-fix Marker

`rick-fix` annotates rows with status markers. Route each finding by what's in its Issue or Fix cells:

| Marker present                        | Comment file destination                                                                                                               |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `~~strikethrough~~` + `✓ FIXED`       | "Verified clean (fixed this round)" — do NOT generate a PR comment                                                                     |
| `⚠ FIX-NEEDS-REVIEW`                  | KEEP in its P-tier comment block, **prefix the body with `⚠ FIX-NEEDS-REVIEW:`** — fix landed but tests failed; reviewer must see this |
| `⊘ SKIPPED: <reason>`                 | KEEP in its P-tier comment block, **append `(engineer skipped: <reason> — your call)`** to the body                                    |
| `BLOCKED` (no row marker — `rick-fix` prints to chat only, doesn't mark the report) | Treat as OPEN. If the `rick-fix` chat transcript is in context, prepend `BLOCKED — <engineer's question>:` to the body so the reviewer can answer it. |
| No marker                             | KEEP in its P-tier comment block as-is — still open                                                                                    |

Re-runnable: run `/rick-fix all`, then `/rick-review-comments` again, and the file shrinks to only the rows that still need a reviewer comment.

## Step 3 — Write `docs/rick/<REVIEW_NAME>/review/current-comments.md`

**Overwrite any existing file** — mirrors how `current.md` gets overwritten per run.

Required sections (skip when empty):

1. **Header** — `# PR #{N} — Inline Comments for Reviewer (v{N})` + one-line context paragraph naming council verdict + open-finding counts + cross-cutting summary. If no PR, lead with the REVIEW_NAME instead of `PR #{N}`. If sibling `*-comments.md` files exist in `docs/rick/<REVIEW_NAME>/review/` (e.g. from an adjacent stacked PR), mention them in a second line.

   Example:

   ```
   # PR #318 — Inline Comments for Reviewer (v3)

   Council verdict NEEDS FIXES — 1 P0, 2 P1, 4 P2 open after `/rick-fix`. The P0 and one P1 are cross-cutting (Rick C-137 + Evil Morty).
   ```

2. **P0 / P1 / P2 / P3 sections** — one per non-empty tier. Each finding renders as a `### \`file:line\`` heading followed by a blockquote body. The body opens with the imperative ask, includes a fenced code block when the fix is more than a few words, and ends with a one-line `why:` citing a peer-pattern, PR body, settled decision, or rule. Flag cross-cutting findings (rows with `**` in the source table) with `cross-cutting (<agent A> + <agent B>)`. Note Step 7.5 demotions inline (`originally MEDIUM; demoted to P3 — regression-catching gap`).

   Example:

   ```
   ### `convex/expenses.ts:42`
   > guard the participant lookup before indexing. participant could be deleted between the read and the update.
   > ```ts
   > const participant = await ctx.db.get(participantId);
   > if (!participant) throw new Error("participant gone");
   > ```
   > why: Pickle Rick flagged the same pattern in `convex/events.ts:88` last review; settled convention is throw-on-missing for derived reads.
   ```

3. **Not commenting (killed at Step 7.5)** — bullet list of kills, each naming which Step 7.5 check killed it (self-defeat / hedge / impact restatement) and the cited phrase that triggered the kill.

   Example:

   ```
   - **`convex/expenses.ts:42` — "validate participantId before lookup."** killed by hedge scan: cited "if a future caller passes a stale id" — speculative, not a real-today defect.
   ```

4. **Verified clean** — bullet list of items the reviewer does NOT need to comment on. Two sources, suffix differs (pick one — don't write the slash literally):
   - Strikethrough/FIXED rows in `current.md` → suffix `**FIXED** (this round)`.
   - Findings present in `v<N-1>.md` but absent from `current.md` → suffix `**FIXED** (carry-over from v<N-1>)`.

   Example:

   ```
   - **`convex/events.ts:88`** — guard the participant lookup before indexing. **FIXED** (this round).
   - **`src/auth/session.ts:14`** — debounce session refresh on focus. **FIXED** (carry-over from v2).
   ```

5. **Order of operations if you only have time for some** — priority-ordered list of OPEN findings only (skip FIXED rows). P0 → P1 → P2 → P3, cross-cutting first within tier. One line per finding: `1. **[P0]** \`file:line\` — <one-line ask>` (suffix cross-cutting rows with ` **`). If zero open findings remain: `nothing left — all findings either fixed or killed. ship it.`

### Before writing the file

Three checks. If any fails, fix the draft before writing — do not ship placeholders.

- Every P-tier finding has (a) a `### \`file:line\`` heading, (b) a one-line `why:`, (c) a fenced code block if the fix is more than a few words.
- Every kill in "Not commenting" names which Step 7.5 check killed it (self-defeat / hedge / impact restatement).
- The "Order of operations" list contains zero FIXED rows. If a strikethrough leaked in, drop it.

## Step 4 — Voice (binding)

- Author voice — fragments OK, lowercase OK, full sentences not required.
- No apologies, no fluff, no hedging in the **ask** (the first line of the body). If the ask contains `consider`, `might want to`, `you may want`, `could potentially`, or `perhaps` — strike the sentence and rewrite as imperative. The `why:` line is a justification, not an ask — leave reasoning verbs there alone.
- If a prior `*-comments.md` exists in `docs/rick/<REVIEW_NAME>/review/`, match its tone exactly. Otherwise the voice above is the convention.

(Structural rules — `why:` line presence, fenced-block presence, kill-reason presence — are enforced by the Step 3 pre-write gate, not duplicated here.)

## Step 5 — Print

```
Comments file written: docs/rick/<REVIEW_NAME>/review/current-comments.md
Open: P0:{n} P1:{n} P2:{n} P3:{n} | Fixed this round:{n} | Carry-overs fixed:{n} | Killed:{n} | Needs-review:{n}
Verdict: {APPROVED|NEEDS FIXES}
```

Then stop. No "want me to paste these into the PR," no "want me to commit," no "want me to ping the reviewer." The file is the deliverable; the reviewer copies from it on their own time.
