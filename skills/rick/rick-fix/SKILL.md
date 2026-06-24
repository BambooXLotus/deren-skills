---
name: rick-fix
description: Resolve findings from a rick-review report sequentially — TDD-route behavioral findings, direct-edit structural ones, mark FIXED / SKIPPED / BLOCKED.
argument-hint: "[<report-path>] <finding-ref> — ref: P0-1 | all-P0 | all-P1 | all (report path auto-resolves from branch if omitted)"
disable-model-invocation: true
---

# rick-fix

Reads a `rick-review` report and implements the findings sequentially. Each finding routes to TDD (`/tdd`) if it describes observable behavior, or a direct surgical fix otherwise. Each fix runs its affected tests before the next one starts.

## Step 1 — Parse Arguments

Split `$ARGUMENTS` on the **last space**. `FINDING_REF` = last token. `REPORT_PATH` = everything before.

Valid refs: `P0-1`, `P1-3`, etc. (specific row), or `all-P0`, `all-P1`, `all-P2`, `all-P3`, `all`.

If `FINDING_REF` is missing but `REPORT_PATH` ends in `.md`, default `FINDING_REF` to `all`.

If `REPORT_PATH` is missing, **auto-resolve from the current branch**:

1. Compute `REVIEW_NAME` per [`../rick-review/RESOLVE-FOLDER.md`](../rick-review/RESOLVE-FOLDER.md).
2. Resolve `REPORT_PATH` in priority order — first that exists wins:
   - **a. Canonical (current).** `docs/rick/<REVIEW_NAME>/review/current.md`.
   - **b. Legacy: artifact-first with folder-prefixed filename.** `docs/rick/reviews/<REVIEW_NAME>/<REVIEW_NAME>-rick-review.md`.
   - **c. Legacy: artifact-first with bare filename.** `docs/rick/reviews/<REVIEW_NAME>/rick-review.md`.
   - **d. Legacy: original code-review suffix.** `docs/rick/reviews/<REVIEW_NAME>/<REVIEW_NAME>-code-review.md`.
   - **e. Legacy: issue-N folder.** If branch matches `^([0-9]+)-`, also try `docs/rick/issue-<n>/review/current.md`, `docs/rick/reviews/issue-<n>/issue-<n>-rick-review.md`, then `docs/rick/reviews/issue-<n>/issue-<n>-code-review.md`.
   - **f. Legacy: pr-N folder.** If a PR is open, also try `docs/rick/pr-<n>/review/current.md`, `docs/rick/reviews/pr-<n>/pr-<n>-rick-review.md`, then `docs/rick/reviews/pr-<n>/pr-<n>-code-review.md`.
   - **g.** If nothing resolved, print usage and stop.
3. If a report was found, default `FINDING_REF` to `all` if missing.
4. Usage on stop:

```
Usage: /rick-fix [<report-path>] <finding-ref>
Examples:
  /rick-fix docs/rick/48-rotate-share-token/review/current.md P0-1
  /rick-fix all-P0
  /rick-fix all
```

## Step 2 — Read and Validate Report

Read `REPORT_PATH`. If missing: print error, stop.

Parse findings tables (P0–P3). Each row: index (`P0-1`, `P1-2`…), file:line, issue, fix suggestion, source agent(s).

Resolve `FINDING_REF`:

- `P0-1` → first row of P0 table (single finding)
- `all-P0` → all rows in P0
- `all` → all rows, ordered P0 → P1 → P2 → P3

Skip any finding already marked `FIXED` (strikethrough or `✓ FIXED` in the Fix cell).

If 0 eligible findings remain: print `Nothing to fix — all targeted findings are already FIXED.` and stop.

## Step 3 — Detect Test Runner

Read `package.json` once. Set `RUNNER`:

- If `vitest` in `devDependencies` or `dependencies` → `RUNNER="npx vitest run"`
- Else if `jest` in either → `RUNNER="npx jest --no-coverage"`
- Else → `RUNNER="npm test --"` (project's default; specs passed as positional args)

Read the project for the test file suffix convention. Check `vitest.config.*` / `jest.config.*` / `package.json#jest` for `testMatch` or `testRegex`. If unclear, default to checking both `<base>.test.ts(x)` and `<base>.spec.ts(x)` patterns in step 3c.

## Step 4 — Fix Loop (sequential)

For each eligible finding, in order:

### 4a. Classify: TDD or direct fix?

Read the finding's issue and fix. Ask one question: **can I write a failing test right now that proves this bug exists?**

- **Yes (behavioral):** missing validation, wrong output, missing code path, incorrect side-effect, off-by-one, error swallowed silently. Route to TDD.
- **No (structural):** formatting, decorator/import order, naming, type-only changes, dead-code removal, missing `readonly`, lint-style stuff. Fix directly.

### 4b. Execute the fix

**TDD route (behavioral):** invoke `/tdd` with the target file from `file:line`, the behavior described in the Issue cell verbatim, and a spec file in the same directory with matching base name. The `/tdd` skill runs its red-green loop. After it completes, treat the result as `FIXED` and proceed to 4c.

If `/tdd` is not installed in the project: fall back to the direct-fix path, but explicitly note in the report that the fix is unverified (no test was written).

**Direct route (structural):** read [FIX-PROTOCOL.md](FIX-PROTOCOL.md) and launch as a `general-purpose` sub-agent with these placeholders replaced:

- `{FINDING_INDEX}` — `P0-1`
- `{FILE_LINE}` — `src/users/user.service.ts:47`
- `{ISSUE}` — issue text from the report
- `{FIX_SUGGESTION}` — fix column from the report
- `{AGENT_SOURCE}` — flagging council agent(s)

The sub-agent returns `FIXED`, `SKIPPED`, or `BLOCKED`.

### 4c. Run affected tests (FIXED only — skip for SKIPPED/BLOCKED)

Parse the `Modified files:` block from the sub-agent response. For each modified path, look for sibling test files:

- `<same-dir>/<same-base>.test.ts(x)`
- `<same-dir>/<same-base>.spec.ts(x)`

Deduplicate. If any test files exist:

```bash
$RUNNER <test1> <test2> ...
```

If tests **pass** → keep `FIXED`. Continue to 4d.

If tests **fail** → result becomes `FIX-NEEDS-REVIEW`. Do NOT attempt to fix the tests. Print the failure output and continue to 4d.

If no test files exist → log a warning, keep `FIXED`, continue to 4d. (Behavioral fixes routed through `/tdd` will always have a spec; this branch fires only for direct-fix or `/tdd`-unavailable cases.)

### 4d. Update report

Edit `REPORT_PATH` in place at the finding's row:

| Result | Report change |
|--------|---------------|
| `FIXED` | Wrap Issue and Fix cells in `~~` strikethrough, append `✓ FIXED` to Fix cell |
| `FIX-NEEDS-REVIEW` | Append `⚠ FIX-NEEDS-REVIEW` to the Issue cell (no strikethrough) |
| `SKIPPED` | Append `⊘ SKIPPED: <one-line reason>` to the Issue cell (no strikethrough) |
| `BLOCKED` | No edit. Hold the question for Step 5 |

`FIXED` row example:

```
| 1 | user.service.ts:47 | ~~Missing readonly on injected dep~~ | ~~Add readonly~~ ✓ FIXED | Rick C-137 |
```

## Step 5 — Print Summary

```
rick-fix complete.

Fixed:           {n}
Skipped:         {n}
Needs review:    {n}
Blocked:         {n}

Report updated: {REPORT_PATH}
```

If any findings were `BLOCKED`, list each one with its question after the summary so Morty can answer and rerun.

Then stop. Don't offer to re-run, don't summarize what was fixed — the report has the diff.
