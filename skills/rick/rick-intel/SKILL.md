---
name: rick-intel
description: Gather research on an ADO work item — read the work item + parent epic, sweep vexis-docs and ADRs, write a per-story intel file to docs/rick/<AB-id>/intel/current.md. Use when the user hands you an AB number to research before planning.
argument-hint: "<AB-id> [--refresh]"
---

# Rick Intel

_burp_ Before you write a line of code, you read the dossier, Morty. This skill reads the ADO work item, climbs to the parent epic, sweeps vexis-docs and ADRs, and writes a story-scoped intel file at `docs/rick/<AB-id>/intel/current.md`.

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
2. Compute paths:
   - Folder: `docs/rick/<AB-id>/intel/`
   - Canonical: `docs/rick/<AB-id>/intel/current.md`
3. Check if canonical exists:
   - Exists, no `--refresh` → print path + the file's `Last gathered:` line + the one-line summary footer, stop. Do not re-fetch.
   - Exists, `--refresh` → load existing findings into memory for diff (see Refresh).
   - Does not exist → fresh gather.
4. Create `docs/rick/<AB-id>/intel/` if missing.

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

In order, stop at first match:

1. `fields.Microsoft.VSTS.Common.AcceptanceCriteria` field on the story.
2. Story description, regex `AC-?\d+` or `Backs AC: (AC-?\d+(?:, AC-?\d+)*)`.
3. Parent epic's AC field — when story says "Backs AC-N of parent", pull AC-N text from parent.

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

Path: `docs/rick/<AB-id>/intel/current.md`. See [OUTPUT.md](OUTPUT.md) for the full section order, finding format, and refresh stale-line format.

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
3. Exactly one match → `bd update <bd-id> --set-metadata research=docs/rick/<AB-id>/intel/current.md`.
4. Zero matches → print: `No bd issue linked. Once claimed: bd update <bd-id> --set-metadata research=docs/rick/<AB-id>/intel/current.md`. Continue.
5. Multiple matches → print candidate ids, do nothing, ask Morty to resolve.

## Confirm and stop

Fresh write:

```
Intel filed for <AB-id>: docs/rick/<AB-id>/intel/current.md
<N> ACs, <N> findings, <N> open questions
Domain anchor: <path> (<N> hits) | Keyword sweep: <N> cross-cutting hits
```

Refresh:

```
Intel refreshed for <AB-id>: docs/rick/<AB-id>/intel/current.md
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
