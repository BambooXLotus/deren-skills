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
- **Carry-overs.** If `docs/rick/<REVIEW_NAME>/review/v<N-1>.md` exists, scan it for finding rows whose `file:line` does NOT appear in `current.md` at all (the diff removed the offending code between versions). Those are carry-overs the reviewer can mark "verified clean." Strikethrough/FIXED rows in `current.md` are classified as "this round" by OUTPUT.md section 4 — do NOT also list them as carry-overs (duplicates the entry in the reviewer's file). If no `v<N-1>.md` exists, carry-overs is 0.

## Step 2.5 — Classify Each Finding by rick-fix Marker

`rick-fix` annotates rows with status markers. Route each finding by what's in its Issue or Fix cells:

| Marker present                        | Comment file destination                                                                                                               |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `~~strikethrough~~` + `✓ FIXED`       | "Verified clean (fixed this round)" — do NOT generate a PR comment                                                                     |
| `⚠ FIX-NEEDS-REVIEW`                  | KEEP in its P-tier comment block, **prefix the body with `⚠ FIX-NEEDS-REVIEW:`** — fix landed but tests failed; reviewer must see this |
| `⊘ SKIPPED: <reason>`                 | KEEP in its P-tier comment block, **append `(engineer skipped: <reason> — your call)`** to the body                                    |
| `BLOCKED` (chat-only, no row marker)  | Treat as OPEN. If the `rick-fix` chat transcript is in context, prefix the body with `BLOCKED — <engineer's question>:` so the reviewer can answer.        |
| No marker                             | KEEP in its P-tier comment block as-is — still open                                                                                    |

Re-runnable: run `/rick-fix all`, then `/rick-review-comments` again, and the file shrinks to only the rows that still need a reviewer comment.

## Step 3 — Write `docs/rick/<REVIEW_NAME>/review/current-comments.md`

**Overwrite any existing file** — mirrors how `current.md` gets overwritten per run.

See [OUTPUT.md](OUTPUT.md) for the full section layout (Header, P0–P3 findings, Not commenting, Verified clean, Order of operations) with worked examples. Read it before drafting the body.

### Before writing the file

Three checks. If any fails, fix the draft before writing — do not ship placeholders.

- Every P-tier finding has (a) a `### \`file:line\`` heading, (b) a one-line `why:`, (c) a fenced code block if the fix is more than a few words.
- Every kill in "Not commenting" names which Step 7.5 check killed it (self-defeat / hedge / impact restatement).
- The "Order of operations" list contains zero FIXED rows. If a strikethrough leaked in, drop it.

## Step 4 — Voice (binding)

- Author voice — fragments OK, lowercase OK, full sentences not required.
- No apologies, no fluff, no hedging in the **ask** (the first line of the body). If the ask contains `consider`, `might want to`, `you may want`, `could potentially`, or `perhaps` — strike the sentence and rewrite as imperative. The `why:` line is a justification, not an ask — leave reasoning verbs there alone.
- If a prior `*-comments.md` exists in `docs/rick/<REVIEW_NAME>/review/`, match its tone exactly. Otherwise the voice above is the convention.

## Step 5 — Print

```
Comments file written: docs/rick/<REVIEW_NAME>/review/current-comments.md
Open: P0:{n} P1:{n} P2:{n} P3:{n} | Fixed this round:{n} | Carry-overs fixed:{n} | Killed:{n} | Needs-review:{n}
Verdict: {APPROVED|NEEDS FIXES}
```

Then stop. No "want me to paste these into the PR," no "want me to commit," no "want me to ping the reviewer." The file is the deliverable; the reviewer copies from it on their own time.
