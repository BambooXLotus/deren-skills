# Comments file output structure (rick-review-comments)

This is what gets written to `docs/rick/<REVIEW_NAME>/review/current-comments.md`.

Required sections (skip when empty).

## 1. Header

`# PR #{N} — Inline Comments for Reviewer (v{N})` + one-line context paragraph naming council verdict + open-finding counts + cross-cutting summary. If no PR, lead with the REVIEW_NAME instead of `PR #{N}`. If sibling `*-comments.md` files exist in `docs/rick/<REVIEW_NAME>/review/` (e.g. from an adjacent stacked PR), mention them in a second line.

Example:

```
# PR #318 — Inline Comments for Reviewer (v3)

Council verdict NEEDS FIXES — 1 P0, 2 P1, 4 P2 open after `/rick-fix`. The P0 and one P1 are cross-cutting (Rick C-137 + Evil Morty).
```

## 2. P0 / P1 / P2 / P3 sections

One per non-empty tier. Each finding renders as a `### \`file:line\`` heading followed by a blockquote body. The body opens with the imperative ask, includes a fenced code block when the fix is more than a few words, and ends with a one-line `why:` citing a peer-pattern, PR body, settled decision, or rule. Flag cross-cutting findings (rows with `**` in the source table) with `cross-cutting (<agent A> + <agent B>)`. Note Step 7.5 demotions inline (`originally MEDIUM; demoted to P3 — regression-catching gap`).

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

## 3. Not commenting (killed at Step 7.5)

Bullet list of kills, each naming which Step 7.5 check killed it (self-defeat / hedge / impact restatement) and the cited phrase that triggered the kill.

Example:

```
- **`convex/expenses.ts:42` — "validate participantId before lookup."** killed by hedge scan: cited "if a future caller passes a stale id" — speculative, not a real-today defect.
```

## 4. Verified clean

Bullet list of items the reviewer does NOT need to comment on. Two sources, suffix differs:

- Strikethrough/FIXED rows in `current.md` → suffix `**FIXED** (this round)`.
- Findings present in `v<N-1>.md` but absent from `current.md` → suffix `**FIXED** (carry-over from v<N-1>)`.

Example:

```
- **`convex/events.ts:88`** — guard the participant lookup before indexing. **FIXED** (this round).
- **`src/auth/session.ts:14`** — debounce session refresh on focus. **FIXED** (carry-over from v2).
```

## 5. Order of operations if you only have time for some

Priority-ordered list of OPEN findings only (skip FIXED rows). P0 → P1 → P2 → P3, cross-cutting first within tier. One line per finding: `1. **[P0]** \`file:line\` — <one-line ask>` (suffix cross-cutting rows with ` **`). If zero open findings remain: `nothing left — all findings either fixed or killed. ship it.`
