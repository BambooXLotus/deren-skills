---
name: rick-review-comments
description: Generate a friendly inline-comments markdown file for the PR reviewer from the latest /rick-review report. Reformats P0/P1/P2/P3 findings as per-line comments with code-block fixes, lists Step 7.5 kills the reviewer should skip, and routes rick-fix markers (FIXED → verified clean; FIX-NEEDS-REVIEW / SKIPPED → annotated). Use when the user asks for a "comments file", "PR comments", "inline comments", "drop-in reviewer comments", or runs /rick-review-comments after a council review.
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

Mirror `rick-review` Step 2 (and `rick-fix` Step 1) exactly — drift breaks the round-trip.

Priority order, first that produces a value:

1. `$ARGUMENTS` (first token, if present).
2. **Branch name (sanitized).** `git branch --show-current`, replace `/` and spaces with `-`. Non-empty and not in `{main, master, develop}` → e.g. `48-rotate-share-token`, `feature-add-payments`.
3. **PR number fallback.** Only when on main/master/develop/detached HEAD AND a PR is open for HEAD (`gh pr view --json number 2>/dev/null`) → `pr-<number>`.
4. **Date fallback.** `review-<YYYY-MM-DD>`.

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

1. **Header** — `# PR #{N} — Inline Comments for Reviewer (v{N})` + one-line context paragraph noting council verdict + open-finding counts + anything cross-cutting. If no PR, lead with the REVIEW_NAME instead of `PR #{N}`. If sibling `*-comments.md` files exist in `docs/rick/<REVIEW_NAME>/review/` (e.g. from an adjacent stacked PR), mention them in a second line.

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

3. **Not commenting (killed at Step 7.5)** — bullet list of kills with the one-line reason (self-defeat / hedge / impact restatement). Format:

   ```
   - **`file:line` — <one-line claim>.** <one-line reason it was killed>.
   ```

4. **Verified clean** — bullet list of items the reviewer does NOT need to comment on. Two sources:
   - Findings from THIS review that `rick-fix` resolved this round (rows with `~~strikethrough~~` + `✓ FIXED`).
   - Carry-overs from prior `v<N-1>.md` that this version resolved.
     Format each as `- **<file:line>** — <one-line summary>. **FIXED** (this round / carry-over).`

5. **Order of operations if you only have time for some** — priority-ordered list of OPEN findings only (skip FIXED rows). P0 → P1 → P2 → P3, cross-cutting first within tier. One line per finding: `1. **[P0]** \`file:line\` — <one-line ask>` (suffix cross-cutting rows with ` **`). If zero open findings remain: `nothing left — all findings either fixed or killed. ship it.`

### Before writing the file

Three checks. If any fails, fix the draft before writing — do not ship placeholders.

- Every P-tier finding has (a) a `### \`file:line\`` heading, (b) a one-line `why:`, (c) a fenced code block if the fix is more than a few words.
- Every kill in "Not commenting" names which Step 7.5 check killed it (self-defeat / hedge / impact restatement).
- The "Order of operations" list contains zero FIXED rows. If a strikethrough leaked in, drop it.

## Step 4 — Voice (binding)

- Author voice — fragments OK, lowercase OK, full sentences not required.
- No apologies, no fluff, no hedging in the **ask** (the first line of the body). If the ask contains `consider`, `might want to`, `you may want`, `could potentially`, or `perhaps` — strike the sentence and rewrite as imperative. The `why:` line is a justification, not an ask — leave reasoning verbs there alone.
- One-line WHY per finding so the reviewer can adjudicate edge cases.
- Fenced code blocks for fixes longer than a few words.
- If a prior `*-comments.md` exists in `docs/rick/<REVIEW_NAME>/review/`, match its tone exactly. Otherwise the voice above is the convention.

## Step 5 — Print

```
Comments file written: docs/rick/<REVIEW_NAME>/review/current-comments.md
Open: P0:{n} P1:{n} P2:{n} P3:{n} | Fixed this round:{n} | Carry-overs fixed:{n} | Killed:{n} | Needs-review:{n}
Verdict: {APPROVED|NEEDS FIXES}
```
