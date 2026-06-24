---
name: rick-plan-improve
description: "Bounded 3-round critique-and-fix loop on a plan written by /rick-plan. Snapshots pre/post versions alongside the canonical. 500-line cap."
argument-hint: "[<folder> | <path-to-plan.md>] — empty resolves to current branch"
disable-model-invocation: true
---

# rick-plan-improve

Bounded improvement loop on a plan file written by `/rick-plan` (or any plan markdown). Reads the canonical plan, critiques against weakness categories, applies surgical edits, snapshots pre and post, up to 3 rounds.

## Resolve the target plan

`$ARGUMENTS` resolution order, use the first that produces a value:

1. **Empty.** Resolve to current branch: `git branch --show-current`, sanitize `/` and spaces to `-`. Target canonical: `docs/rick/<branch>/plan/current.md`.
2. **Path ending in `.md`.** Treat as literal path. Target canonical = that path. `<plan-dir>` = its parent directory.
3. **Otherwise.** Treat as a folder name. Target canonical: `docs/rick/<arg>/plan/current.md`.

If the resolved canonical does not exist, stop with `No plan at <path>. Run /rick-plan first.` Do not silently fall through.

## Before the loop

1. Read the canonical plan file. The actual file, not your memory of it.
2. Check for an intel dossier per [`../rick-intel/LOOKUP.md`](../rick-intel/LOOKUP.md). If one exists, read its Story Context + Findings — they're trust sources for verifying plan claims in step 6 and for spotting **Unverified claim** weakness. Plan and intel divergence (plan asserts X, intel cites Y) is a finding in the round.
3. Read `CLAUDE.md` (or equivalent project instructions). Half of plan problems are specs that violate conventions already written down six inches from Morty's face.
4. Determine versioning state:
   - **Version directory:** `<plan-dir>/` (same dir as the canonical — v1.md, v2.md, etc. sit alongside current.md).
   - **Next version `N`:** list `<plan-dir>/v*.md`, find the highest integer (ignore `-pre` suffix), next = highest + 1. A canonical written by `/rick-plan` always seeds `v1.md`, so the first improve run is `v2`.
   - **Snapshot pre:** copy the canonical plan → `<plan-dir>/v<N>-pre.md` BEFORE any edits. This is the rollback point. No exceptions.
5. Read `<plan-dir>/VERSIONS.md` if it exists. Records prior changes and rationale. Context, not a shield. To reverse a prior decision: state it, cite evidence its rationale is wrong, explain why the evidence invalidates it. All three or leave it alone.
6. Verify claims in the plan against local files and the intel dossier (if present). Skip external URLs unless the claim cannot be verified locally.
7. Count body lines (excluding frontmatter). Record as the baseline for the size stop condition.

## Stop conditions (first that fires, stop immediately)

1. **3 rounds completed.** Hard cap. No exceptions.
2. **Round produces zero net-new changes.** Say "no improvements found this round" and stop.
3. **Plan exceeds 500 lines.** Round must cut before it can add. If you cannot get under 500, stop and tell Morty the plan needs to be split.

## Loop protocol (up to 3 rounds)

Edit the canonical plan file directly. The pre-loop backup is already safe in `<plan-dir>/v<N>-pre.md`.

Each round:

1. **Read the canonical plan.** The actual file. Not your memory from the last round.
2. **Critique against the weakness categories in [WEAKNESS-CATEGORIES.md](WEAKNESS-CATEGORIES.md).** Read that file once before round 1; carry the categories across rounds. Cite the section, line, or claim that is broken. No citation, no finding. If a section is solid, say so and move on.
3. **Rank worst first.** Which gap costs the most wasted engineering time or produces the worst production bug? Name the failure mode in one sentence before fixing it. If you cannot name it, the finding does not qualify.
4. **Apply surgical fixes.** Do not rewrite sections that work.
5. **Count body lines after edits.** If over 500, stop the loop immediately.
6. **Report what changed.** Format: `- [Section] Change description (why: failure it prevents)`. Hold these for the post-loop VERSIONS.md entry.

## After the loop

1. Copy the final plan → `<plan-dir>/v<N>.md` with YAML frontmatter prepended:

   ```yaml
   ---
   version: <N>
   date: <YYYY-MM-DD>
   author: rick-plan-improve
   parent: v<N>-pre.md
   summary: <one-line stop-condition + total change count>
   ---
   ```

2. Append a full entry to `<plan-dir>/VERSIONS.md` (create with the header below if missing — but normally `/rick-plan` already created it):

   ```markdown
   # Plan Versions: <folder>

   Append-only history. v1 is the initial plan; later versions are /rick-plan-improve runs.
   ```

   ```
   - v<N> | <YYYY-MM-DD> | <stop-condition: 3-rounds | zero-changes | over-cap>
     - Round 1: <one line, what changed>
     - Round 2: <one line, or "stopped early">
     - Round 3: <one line, or "stopped early">
   ```

3. Print exactly:

   ```
   Improved <folder> (v<N>): <stop-condition>. <R> rounds, <C> total changes. Canonical: <plan-path>. History: <plan-dir>/VERSIONS.md
   ```

   Where `<folder>` is the grandparent directory name (e.g. for `docs/rick/48-rotate-share-token/plan/current.md`, `<folder>` is `48-rotate-share-token`).

Then stop. Don't recap. Don't offer to revert. The history file has the diff.

## What NOT to do

- Don't add sections for hypothetical future requirements. Every section must prevent a specific, nameable failure.
- Don't expand the plan into a tutorial or add structure for its own sake. Engineers can read code.
- Don't soften language. "This will cause silent data loss" beats "this could potentially lead to inconsistencies."
- Don't touch `v*.md` snapshot files mid-loop. Only write pre (before round 1) and post (after the final round).
- Don't run the loop on `rick-plan-improve` itself. That's `rick-improve`'s job.

## Stop condition

After the post snapshot and VERSIONS.md update, print the one-line summary and stop. No recap, no "want me to revert," no offer to iterate further. The history file has the diff.
