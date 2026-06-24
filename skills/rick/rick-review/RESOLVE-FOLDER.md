# Resolve `REVIEW_NAME`

The folder identity for a feature thread. `/rick-plan`, `/rick-save`, `/rick-review`, `/rick-fix`, `/rick-review-comments`, and `/rick-recap` all land artifacts under `docs/rick/<REVIEW_NAME>/`, so the same branch produces the same folder name across every skill in the loop.

## Priority order (first that produces a value wins)

1. **Branch name (sanitized).** `git branch --show-current`, replace `/` and spaces with `-`. Non-empty and not in `{main, master, develop}` → e.g. `48-rotate-share-token`, `feature-add-payments`. This is the normal path.
2. **PR number.** Only when on main / master / develop / detached HEAD AND a PR is open for HEAD (`gh pr view --json number 2>/dev/null`) → `pr-<number>` (e.g. `pr-213`).
3. **Date fallback.** `review-<YYYY-MM-DD>` when nothing else resolves.

Callers that accept an explicit folder override in `$ARGUMENTS` (e.g. `/rick-review-comments 48-rotate-share-token`) consume the argument before this ladder fires.
