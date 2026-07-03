---
name: rick-intel
description: Gather research on an ADO work item — read the work item + parent epic, sweep vexis-docs and ADRs, write a per-story intel file to docs/rick/<folder>/intel/current.md. Use when the user hands you an AB number to research before planning.
argument-hint: "<AB-id> [--refresh]"
---

# Rick Intel

_burp_ Before you write a line of code, you read the dossier, Morty. This skill reads the ADO work item, climbs to the parent epic, sweeps vexis-docs and ADRs, and writes a story-scoped intel file at `docs/rick/<folder>/intel/current.md`.

Sister of `rick-plan`. Plan tells you what to build. Intel tells you what's already known. Run intel first.

## Argument

Required. The ADO work item id. Accepts `AB16289`, `16289`, or `ab16289`. Normalize to `AB<digits>`.

If `$ARGUMENTS` is empty or no AB-id can be parsed, print one line and stop:

```
Fine, Morty, which story. Pass an ADO id.
```

Optional `--refresh` anywhere in `$ARGUMENTS` triggers re-gather (see Refresh).

## Intro

Pick one line at random from [INTROS.md](INTROS.md), substitute `<AB-id>`, print it. Then proceed.

## Pre-flight

1. Normalize the AB-id.
2. Resolve the folder. Intel is almost always run pre-branch (the whole point is research before you build), so the folder name should match **the branch the user will create after reading**. Priority order, first that produces a value:
   1. **Branch-already-cut override.** `git branch --show-current` returns a feature branch (not in `{main, master, develop}` and non-empty) **AND the branch name contains the AB-id digits** (e.g. branch `feat/15920-data-gap` for AB15920) → sanitize (`/` and spaces → `-`) and use it. The digit-substring check is mandatory — without it, running `/rick-intel AB15920` while sitting on `feature/WP-FE7-Template-Management` lands the dossier inside the unrelated feature's folder. If you're on a feature branch for a *different* story, priority 1 doesn't fire.
   2. **Predict the branch name.** Fetch ADO title cheaply:
      ```bash
      az boards work-item show --id <digits> --org https://dev.azure.com/VizyFintech --query 'fields."System.Title"' -o tsv
      ```
      Slugify per `rick-plan/PICK-FOLDER.md` step 2: truncate at the first em-dash / en-dash / ` - `; lowercase; replace non-`[a-z0-9]` runs with `-`; trim trailing hyphens; cap at 40 chars without cutting mid-segment. Result: `<digits>-<title-slug>` (e.g. `16289-rotate-share-link-ui`).
   3. ADO title fetch failed → fall back to `ab<digits>` (e.g. `ab16289`) and record an open question.
3. Compute paths:
   - Folder: `docs/rick/<folder>/intel/`
   - Canonical: `docs/rick/<folder>/intel/current.md`
4. **Glob check for prior intel** at this AB-id (handles the case where intel was written before but the folder name has since drifted, e.g. a different branch was created later): `find docs/rick -type f -path '*/intel/current.md' -exec grep -l '<AB-id>' {} \;`. If a single match exists at a different path, treat it as the canonical instead (don't write a second copy under the predicted name). If multiple match, print them and stop — Morty resolves.
5. Check if canonical exists:
   - Exists, no `--refresh` → print path + the file's `Last gathered:` line + the one-line summary footer, stop. Do not re-fetch.
   - Exists, `--refresh` → load existing findings into memory for diff (see Refresh).
   - Does not exist → fresh gather. Cache the title from step 2 so the gather phase doesn't re-fetch it.
6. Create `docs/rick/<folder>/intel/` if missing.

## Gather (no code, info only)

Source order: ADO → vexis-docs → ADRs. Record any failure as an open question and keep going.

### 1. ADO work item

```bash
az boards work-item show --id <digits> --org https://dev.azure.com/VizyFintech
```

Extract:

- `fields.System.Title` → story title
- `fields.System.Description` → raw description (HTML — strip tags for Scope prose, keep plaintext for AC parsing)
- `fields.Microsoft.VSTS.Common.AcceptanceCriteria` → AC field if present
- `fields.System.State`, `fields.System.IterationPath`, `fields.System.AssignedTo.displayName`
- `relations[?attributes.name=='Parent']` → parent work item URL → parent id
- `relations[?attributes.name=='GitHub Pull Request']` → PR refs (extract last URL segment as PR number)

If `az` fails or unauthenticated, record open question "ADO unauthenticated — run `az login` and `--refresh`" and continue with whatever else is reachable.

### 2. ADO parent epic

For the parent id from step 1, run `az boards work-item show` again. Extract title and AC. Use to derive the domain anchor and to quote parent AC text when the story Backs AC-N of parent.

### 3. AC parsing

Try each source in order, stop at first that yields ACs:

1. `fields.Microsoft.VSTS.Common.AcceptanceCriteria` field on the story.
2. Story description body — scan the HTML-stripped description for AC content:
   - "Acceptance Criteria" heading (any level, case-insensitive; also "AC", "Criteria") followed by a list.
   - `AC-?\d+[:.\)]` markers with the prose that follows.
   - Gherkin-style `Given … When … Then …` blocks.
3. Parent epic's AC field — when story description references AC-N (regex `AC-?\d+` or `Backs AC: (AC-?\d+(?:, AC-?\d+)*)`), pull the referenced text from parent.

No AC parseable → file uses one `## Findings` flat section instead of per-AC sections. Everything else is unchanged.

### 4. Domain anchor

Derive candidate tokens from parent title + story title:

- Lowercase, tokenize on spaces, `|`, `-`, `/`.
- Drop noise: `BE`, `FE`, `WP`, digits, `service`, `module`, `task`, `bug`, `epic`, `story`, generic verbs.

Find docs root, first that exists wins:

1. `vexis-docs/` in current repo
2. `../vexis-docs/` sibling repo
3. `docs/` fallback

For each token: `find <docs-root> -type d -iname '*<token>*'`. First directory match wins. Full read of that subtree.

No match → record open question "No domain anchor matched for tokens: …". Continue with sweep only.

### 5. Keyword sweep

Extract sweep terms from story title + Scope text + AC titles. Drop noise tokens.

For each term: `grep -rli --include='*.md' <term> <docs-root> docs/adr/ 2>/dev/null`.

Dedupe hits against the domain-anchor read. For each unique hit: open the file, find the first paragraph containing the term, summarize in ≤2 sentences. Finding text = that summary. Source = `<path>:<line>` of the matched paragraph.

Terms with zero hits → open question "No vexis-docs match for `<term>`".

### 6. ADRs

`docs/adr/` (or `vexis-docs/adr/` if exists). Same sweep over titles and bodies. Same finding format.

## Write the file

Path: `docs/rick/<folder>/intel/current.md`. See [OUTPUT.md](OUTPUT.md) for the full section order, finding format, and refresh stale-line format.

## Refresh

When `--refresh` is passed and the file exists:

1. Read `current.md`, parse every `- … — Source: <S> — Found: <D>` line into a map keyed by source.
2. Run the full gather phase fresh.
3. For each gathered finding:
   - Same source, summary matches → keep existing line untouched.
   - Same source, summary differs → strike old, append new with today.
   - New source → append new with today.
4. For each existing source not present in fresh gather → strike old with today as Stale.
5. Update `Last gathered:` to today.
6. Rewrite the file.

Never delete a finding line. Stale markers are history.

## ADO writeback

After writing the file, post an HTML comment to the ADO work item. Body shape + idempotency rule live in [ADO-COMMENT.md](ADO-COMMENT.md). Substitute `<AB-id>`, `<N>`, and `<YYYY-MM-DD>` before posting.

Failure → don't block file write. Print:

```
Warning: ADO writeback failed: <reason>
Run manually: az boards work-item update --id <digits> --discussion '<html-body>'
```

## bd writeback

After writing the file, attempt bd linkage.

1. `bd list --json 2>/dev/null` — bd absent → skip silently.
2. Find issue where `metadata.ado == <id-numeric>`.
3. Exactly one match → `bd update <bd-id> --set-metadata research=docs/rick/<folder>/intel/current.md`.
4. Zero matches → print: `No bd issue linked. Once claimed: bd update <bd-id> --set-metadata research=docs/rick/<folder>/intel/current.md`. Continue.
5. Multiple matches → print candidate ids, do nothing, ask Morty to resolve.

## Confirm and stop

Fresh write:

```
Intel filed for <AB-id>: docs/rick/<folder>/intel/current.md
<N> ACs, <N> findings, <N> open questions
Domain anchor: <path> (<N> hits) | Keyword sweep: <N> cross-cutting hits
```

If the folder was *predicted* (Pre-flight step 2 priority 2 — no feature branch was checked out) AND the predicted name differs from the current branch, follow up with one line so Morty can branch into the dossier:

```
Next: git checkout -b <folder>
```

Refresh:

```
Intel refreshed for <AB-id>: docs/rick/<folder>/intel/current.md
<N> carried, <M> updated/stale-marked, <K> new
```

If ADO writeback warned, print the warning line first.

## Rules

- No code. No implementation suggestions. Pure research synthesis.
- Cite every finding. Path-line or URL. No bare claims.
- Verbatim ADO description in Scope — strip HTML, don't paraphrase.
- Stale-mark, never delete.
- Idempotent. Two runs without `--refresh` should be free: read, print, exit. Don't re-fetch ADO unprompted.
- _burp_ Don't suit up twice for the same dimension, Morty.
