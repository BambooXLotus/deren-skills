---
name: rick-plan-improve
description: "Bounded 3-round improvement loop on an existing implementation plan (typically one written by /rick-plan). Takes a folder name, branch name, or explicit path; snapshots pre/post versions alongside the canonical at docs/rick/<folder>/plan/, critiques against weakness categories (specification gaps, engineering hazards, convention violations, document quality), applies surgical edits, stops on 3 rounds / zero changes / 500-line cap. Use when user invokes /rick-plan-improve, says \"improve this plan\", \"tighten the plan\", or wants to refine an existing plan without rewriting from scratch."
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
2. Read `CLAUDE.md` (or equivalent project instructions). Half of plan problems are specs that violate conventions already written down six inches from Morty's face.
3. Determine versioning state:
   - **Version directory:** `<plan-dir>/` (same dir as the canonical — v1.md, v2.md, etc. sit alongside current.md).
   - **Next version `N`:** list `<plan-dir>/v*.md`, find the highest integer (ignore `-pre` suffix), next = highest + 1. A canonical written by `/rick-plan` always seeds `v1.md`, so the first improve run is `v2`.
   - **Snapshot pre:** copy the canonical plan → `<plan-dir>/v<N>-pre.md` BEFORE any edits. This is the rollback point. No exceptions.
4. Read `<plan-dir>/VERSIONS.md` if it exists. Records prior changes and rationale. Context, not a shield. To reverse a prior decision: state it, cite evidence its rationale is wrong, explain why the evidence invalidates it. All three or leave it alone.
5. Verify claims in the plan against local files. Skip external URLs unless the claim cannot be verified locally.
6. Count body lines (excluding frontmatter). Record as the baseline for the size stop condition.

## Stop conditions (first that fires, stop immediately)

1. **3 rounds completed.** Hard cap. No exceptions.
2. **Round produces zero net-new changes.** Say "no improvements found this round" and stop.
3. **Plan exceeds 500 lines.** Round must cut before it can add. If you cannot get under 500, stop and tell Morty the plan needs to be split.

## Loop protocol (up to 3 rounds)

Edit the canonical plan file directly. The pre-loop backup is already safe in `<plan-dir>/v<N>-pre.md`.

Each round:

1. **Read the canonical plan.** The actual file. Not your memory from the last round.
2. **Critique against the weakness categories below.** Cite the section, line, or claim that is broken. No citation, no finding. If a section is solid, say so and move on.
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

## Weakness categories

Every finding must cite one.

### Specification gaps

- **Missing error path.** Plan specifies happy path but not what happens when an outbound call times out, returns garbage, or sends a 200 with an error body. Every outbound call needs: exception thrown, what gets logged, what the caller sees.
- **Ambiguous type.** Field described as "string" when it matters whether it's ISO 8601, Unix epoch, nullable, or an enum. `Record<string, unknown>` needs a comment: intentional or lazy?
- **Unverified claim.** Plan says "the API returns X" with no spec line, response sample, or test result cited. Claims without evidence become bugs when the API returns Y.
- **Missing validation.** Input crossing a trust boundary with no validation specified. What are the constraints? What happens on violation?
- **Implicit dependency.** Plan assumes a service, config value, schema field, or table exists without checking it actually does in the codebase.

### Engineering hazards

- **Wrong implementation order.** Step 3 depends on step 5. Dependency order must be explicit and acyclic.
- **Untestable spec.** Behavior described with no test case, or a test case that does not verify what it claims to. Every non-trivial behavior needs a test name: `should [behavior] when [condition]`.
- **Missing concurrency / race condition.** Two callers hit shared mutable state and the plan does not say how conflicts are handled.
- **Config without defaults or validation.** Env var mentioned with no: required/optional flag, default value, validation rule, or startup behavior if missing.
- **Scope leak.** Plan says "out of scope" but describes behavior that requires it.
- **Scope collapse.** Critique removes a deliverable (doc, code, test) without verifying the underlying work is still covered elsewhere. Changing the *format* of a deliverable is valid. Removing it without redirecting the work is not. Before killing a deliverable, answer: "does the work this represented still need to happen? If so, where?"

### Convention violations

- **CLAUDE.md violation.** Plan contradicts a rule in `CLAUDE.md` or equivalent project instructions. Check naming, file structure, branching rules, test discipline, design-system usage.
- **Pattern divergence.** Plan invents a pattern when one already in the codebase solves the same problem.
- **Missing audit trail.** Plan creates or modifies records that need tracking (createdBy, updatedAt, soft-delete columns) without addressing them.

### Document quality

- **Redundant sections.** Two sections say the same thing. Engineers read one, miss the nuance in the other, build something satisfying neither.
- **Stale reference.** A file path, class name, or method name cited in the plan does not exist in the current codebase.
- **Missing decision rationale.** Design choice with no reason given. Engineers who don't know why will revert it later.

## What NOT to do

- Don't add sections for hypothetical future requirements. Every section must prevent a specific, nameable failure.
- Don't expand the plan into a tutorial or add structure for its own sake. Engineers can read code.
- Don't soften language. "This will cause silent data loss" beats "this could potentially lead to inconsistencies."
- Don't touch `v*.md` snapshot files mid-loop. Only write pre (before round 1) and post (after the final round).
- Don't run the loop on `rick-plan-improve` itself. That's `rick-improve`'s job.

## Stop condition

After the post snapshot and VERSIONS.md update, print the one-line summary and stop. No recap, no "want me to revert," no offer to iterate further. The history file has the diff.
