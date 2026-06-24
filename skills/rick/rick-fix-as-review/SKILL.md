---
name: rick-fix-as-review
description: Convert /rick-fix's local edits into a PENDING GitHub PR review with one-click suggestion blocks in neutral reviewer voice.
argument-hint: "[<REVIEW_NAME>] [<PR_NUMBER>] — both auto-resolve from current branch + gh pr view if omitted"
disable-model-invocation: true
---

# rick-fix-as-review

Bridges `/rick-fix` → GitHub pending PR review.

## Quick start

```
/rick-fix-as-review                        # auto-resolve REVIEW_NAME from current branch, PR from gh pr view
/rick-fix-as-review 48-rotate-share-token  # explicit folder
/rick-fix-as-review 48-rotate-share-token 335  # explicit folder + PR number
```

## Step 1 — Resolve inputs

`REVIEW_NAME`: if `$ARGUMENTS` first token is non-empty and matches `^[a-z0-9][a-z0-9-]*$`, use it. Otherwise resolve per [`../rick-review/RESOLVE-FOLDER.md`](../rick-review/RESOLVE-FOLDER.md).

`PR_NUMBER` + `HEAD_SHA` + `BASE_REF`: if `$ARGUMENTS` second token is numeric, use it as `PR_NUMBER`. Otherwise read from `gh pr view --json number,headRefName,baseRefName,headRefOid 2>/dev/null`.

Verify:

- `docs/rick/<REVIEW_NAME>/review/current.md` exists. If missing: `No canonical review at <path>. Run /rick-review first.` Stop.
- `gh pr view` returned a PR on the current branch. If missing: print usage, stop.

## Step 2 — Pre-flight (mandatory, refuse on failure)

**Check A — working tree must NOT be clean yet.** `git status --short`. If empty: `Working tree is clean. /rick-fix didn't run or already reverted. Nothing to snapshot — stop.`

**Check B — report must show FIXED markers.** `grep -c '✓ FIXED' docs/rick/<REVIEW_NAME>/review/current.md`. If zero: `Report has no ✓ FIXED markers. Run /rick-fix first.`

## Step 3 — Snapshot the work

Land the patch + new files under `docs/rick/<REVIEW_NAME>/review/` so they survive the Step 5 revert.

```bash
# Patch of all modified tracked files
git diff > docs/rick/<REVIEW_NAME>/review/proposed-fixes.patch

# New untracked files (typically new spec files from TDD)
mkdir -p docs/rick/<REVIEW_NAME>/review/proposed-specs
for f in $(git ls-files --others --exclude-standard); do
  cp "$f" "docs/rick/<REVIEW_NAME>/review/proposed-specs/$(basename "$f")"
done
```

Print: `Snapshot saved: docs/rick/<REVIEW_NAME>/review/proposed-fixes.patch + proposed-specs/`.

## Step 4 — Capture post-fix line content

Suggestion blocks mirror the AFTER state, so capture it before Step 5 reverts the tree.

For each row marked `✓ FIXED`:

- Extract `file:line` from the row's File:Line column.
- Read the file at line ± 5 lines from the current (post-fix) working tree.
- Record in memory: `{ finding_index, file_path, line, after_content }`.

## Step 5 — Revert the working tree

```bash
# Revert modified tracked files
git checkout -- $(git diff --name-only)

# Delete untracked files (the ones copied in Step 3)
for f in $(git ls-files --others --exclude-standard); do
  rm "$f"
done

git status --short  # must be empty
```

If `git status --short` is non-empty: `Revert incomplete: <files>. Aborting.` Stop.

## Step 6 — Build the JSON payload

One `comments[]` entry per FIXED finding. Anchored to an in-diff line. Internal review machinery stripped from every body.

### Voice rules (strict)

Before writing each comment body, scrub every forbidden token:

- **Agent names:** `Rick`, `Morty`, `Pickle Rick`, `Scientist Rick`, `Evil Morty`, `Doofus Rick`, `Smart Morty`, `Rick C-137`, `Rick Prime`, `Council`, `council`.
- **Versions:** `v1`, `v2`, ..., `vN`, `version N`.
- **Process jargon:** `carry-over`, `settled`, `cross-cutting`, `Total Rickall`, `parasite`, `skeptic`, `kill`, `demote`, `strengthen`, `Step 7.5`, `Step 7.7`.
- **Severity ladder:** `P0`, `P1`, `P2`, `P3`.
- **Internal markers:** `**` cross-cutting flags, `✓ FIXED`, `⊘ SKIPPED`, `⚠ FIX-NEEDS-REVIEW`.

Replace with neutral reviewer voice:

- State the issue plainly.
- Give the fix (suggestion block where possible).
- One-line `why:` citing a peer-pattern, settled decision, or rule. No fluff, no apologies, no "you might want to consider."
- Fragments fine, lowercase fine.

Top-level review body: **≤ 2 sentences**, neutral. Example: `Few small things from a review pass — one validation-error field that gets stripped from the response, a dead symbol, and two spec hardenings. Suggestion blocks below have one-click commit where they fit.`

### Anchoring rules

Each comment attaches to a line GitHub considers "in the diff." Decide per finding:

1. **Net-new file in this PR** — any line is anchorable. Use the cited line directly.
2. **Modified file, cited line within a diff hunk (or 3-line context expansion)** — use the cited line.
3. **Modified file, cited line OUTSIDE every hunk** — re-anchor on the nearest in-hunk line. Add `Fix at line N below` to the body.
4. **Target file is on the base branch (not in this PR's diff at all)** — anchor on a *producer* line in another file that IS in the diff (e.g. a DTO fix where the DTO is on base anchors on a route file that emits the missing field). Body explains where the actual edit lands.

Verify hunk coverage with `git diff origin/$BASE_REF...HEAD -- <file> | grep -E '^@@'` before assuming a line is anchorable.

### Suggestion block construction

| Situation | Block shape |
|---|---|
| Single-line replacement | Single `` ```suggestion `` block with the new line content |
| Contiguous multi-line replacement | `start_line` + `line` + `start_side: RIGHT` + `side: RIGHT`; suggestion block contains the full replacement range |
| Line deletion | Empty `` ```suggestion\n``` `` block |
| Net-new file content / non-contiguous edits / fix on a different file than the anchor | No `suggestion` block — use a `` ```typescript `` fenced block + plain markdown explaining where to apply |

Single-line shape:

````json
{
  "path": "src/path/to/file.ts",
  "line": 42,
  "side": "RIGHT",
  "body": "<neutral reviewer comment>\n\n```suggestion\n<after content>\n```"
}
````

Multi-line shape:

```json
{
  "path": "...",
  "start_line": 307,
  "line": 311,
  "start_side": "RIGHT",
  "side": "RIGHT",
  "body": "..."
}
```

### Full payload shape

```json
{
  "commit_id": "<HEAD_SHA from Step 1>",
  "body": "<≤ 2-sentence neutral summary>",
  "comments": [ ... ]
}
```

Each `comments[]` entry uses the single-line or multi-line variant above.

**Do NOT include `event`.** Omitting `event` creates a PENDING review (invisible to the PR author until the user submits in the UI). Setting `event: ""` returns 422 — omit the key entirely.

Write the payload to `/tmp/rr-as-review-<PR_NUMBER>.json`.

## Step 7 — Post the review

```bash
OWNER_REPO=$(gh repo view --json owner,name --jq '.owner.login + "/" + .name')
gh api repos/$OWNER_REPO/pulls/<PR_NUMBER>/reviews \
  --method POST \
  --input /tmp/rr-as-review-<PR_NUMBER>.json
```

**Handle 422s.** GitHub returns `Line could not be resolved` when an anchored line is outside the diff hunks. The error doesn't say which line.

1. Re-check each anchor's line number against the hunk headers: `git diff origin/$BASE_REF...HEAD -- <file> | grep -E '^@@'`.
2. For any failure: re-anchor per the Step 6 rules.
3. Retry the POST.

Maximum 3 retry rounds. If still failing, fall back to file-level: anchor every failing comment on the first in-diff line of its file with `Fix at line N` in the body.

## Step 8 — Voice audit (mandatory before printing success)

Grep the posted payload for forbidden tokens:

```bash
grep -iE 'rick|morty|council|carry-over|settled|cross-cutting|total rickall|\bp[0-3]\b|\bv[1-9]\b|step 7\.5|step 7\.7|parasite|skeptic|✓ fixed' /tmp/rr-as-review-<PR_NUMBER>.json
```

If anything matches: the language is contaminated. Delete the pending review:

```bash
gh api repos/$OWNER_REPO/pulls/<PR_NUMBER>/reviews/<review_id> --method DELETE
```

Rewrite the offending comments in neutral voice, re-post. Do not surface success until the grep is clean.

## Step 9 — Print the result

```
Pending review posted: <review_html_url>
Comments: <n> | PR: #<PR_NUMBER>
Artifacts: docs/rick/<REVIEW_NAME>/review/proposed-fixes.patch + proposed-specs/

Open the URL → "Files changed" → pending review banner. Read through, edit any comments inline, then submit (Comment / Approve / Request changes).
```

## Rollback

Abandon the workflow after posting:

```bash
gh api repos/$OWNER_REPO/pulls/<PR_NUMBER>/reviews/<review_id> --method DELETE
```

Works because the review is PENDING. Once you submit in the UI, deletion requires `dismiss-review` instead.

Restore the fixes to the local working tree:

```bash
git apply docs/rick/<REVIEW_NAME>/review/proposed-fixes.patch
cp docs/rick/<REVIEW_NAME>/review/proposed-specs/* <matching src/ paths>
```
