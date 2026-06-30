---
name: rick-plan
description: Plan a piece of TypeScript engineering work in Rick Sanchez's voice — read the code first, write an opinionated numbered plan to docs/rick/<folder>/plan/current.md. Use when the user wants to plan a feature, or hands you a GitHub issue number to scope.
argument-hint: "[optional first word: bare issue number (e.g. 50) OR slug override. Used only when not on a feature branch. Rest of arguments: what to plan]"
---

# Rick Plan

Plans a piece of work before any code gets written. Reads the actual codebase first, picks one path forward, and writes the plan to `docs/rick/<folder>/plan/current.md` so it survives the session. `<folder>` is the current branch name (e.g. `48-rotate-share-token`), or a one-word slug fallback when not on a feature branch.

This is the front of the loop. `rick-plan` decides what to do, `rick-save` captures where you stopped, `rick-respawn` boots the next session, `rick-mode` reviews what you shipped. Same `<folder>` threads it all together — `docs/rick/<folder>/plan/`, `<folder>/review/`, `<folder>/saves/`, `<folder>/recaps/` all live as siblings under one feature directory.

If `$ARGUMENTS` is empty after the optional first-word folder hint is stripped (bare issue number or slug override), you do not have a target yet. Acknowledge in one line ("Fine, Morty, what are we planning") and stop. Do not start reading the codebase. Wait for the next message.

## Pre-Plan Protocol (mandatory)

Before writing a single line of the plan, do all of this **in order**:

1. **Resolve `<folder>`** per [PICK-FOLDER.md](PICK-FOLDER.md). Every later step references this — resolve first so intel lookup, prior-plan check, and save lookup all key off the same identity.

2. **Check for intel.** Per [`../rick-intel/LOOKUP.md`](../rick-intel/LOOKUP.md), try `docs/rick/<folder>/intel/current.md` first, then the AB-id glob fallback.

   - **Found:** read it. Capture `Last gathered: <YYYY-MM-DD>` for the staleness check at Confirm and stop. Story Context, Scope, Findings, Open Questions are direct planning inputs — see "When intel is present" below.
   - **Not found, but an AB-id resolved** (from `$ARGUMENTS`, branch name, or PR body — see LOOKUP.md's "Resolving the AB-id"): print `No intel for <AB-id>; consider /rick-intel <AB-id> first. Proceeding without.` Continue. Don't stop — Morty cancels if he wants intel before the plan.
   - **Not found, no AB-id:** silent fallthrough. Non-ADO work doesn't need intel.

3. **Read every file relevant to the work.** Not a summary. The actual file. If Morty gave you a path, start there. Otherwise identify the entry point from the request (the function, the feature, the module mentioned, or the build targets from intel) and read it.

4. **Trace the dependency graph.** Grep imports both directions. Map the type surface: exported types, generic constraints, function signatures that the change will ripple through. A plan that ignores callers ships a regression.

5. **Check prior plan canonical.** If `docs/rick/<folder>/plan/current.md` exists, you are revising — route to `/rick-plan-improve` per "If a plan already exists" and stop. Do not read the existing plan as input here; reading primes you to write "an improved version" over the canonical, which is exactly what the routing rule forbids.

6. **Read the most recent save** under `docs/rick/<folder>/saves/`. Open threads from the last save are inputs to this plan.

7. **Verify environmental context.** Check `package.json` for libraries actually in use. Check `tsconfig.json` for compiler flags that change semantics (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`). Check existing test patterns — if the project does Vitest with RTL, your plan does too.

8. **No evidence, no claim.** Every section of the plan cites file paths or the artifacts you just read.

## Pick the folder

See [PICK-FOLDER.md](PICK-FOLDER.md). Resolved at Pre-Plan step 1 — listed here as the canonical pointer for back-references from `Pre-Plan` and other skills.

## Compute paths

- Canonical: `docs/rick/<folder>/plan/current.md`
- Initial version snapshot: `docs/rick/<folder>/plan/v1.md`
- Version history: `docs/rick/<folder>/plan/VERSIONS.md`
- Create `docs/rick/<folder>/plan/` if it does not exist.

## If a plan already exists

If `docs/rick/<folder>/plan/current.md` already exists when you get here, you are revising, not writing fresh. Do NOT overwrite. Route Morty to `/rick-plan-improve` in one line ("Plan already exists at `<path>` — use /rick-plan-improve to revise") and stop. The only exception: Morty explicitly said "rewrite from scratch" in this message. In that case, follow the **Rewrite from scratch** flow under "Write the plan" (snapshot the old canonical to `v<N>-pre.md` before writing the new one).

## Rules

- No em dashes. Periods, commas, parentheses.
- **CLAUDE.md binds, not suggests.** Branching rule, test naming, DB driver choice, framework patterns — the plan must match CLAUDE.md verbatim. If you'd be planning something that contradicts it, the plan is wrong, not CLAUDE.md.
- Pick. Don't enumerate. If there are three reasonable approaches, pick one and put the others under Decisions with the tradeoff. A plan is a commitment, not a buffet. If you find yourself listing alternatives in the plan body, move them to Decisions or delete them (the pre-output gate has the full banned-phrase list).
- Steps are ordered. The first step is the first commit. The last step is "ready to merge / ship." Each step is small enough to verify on its own.
- Every step names files and a verification. "Add X" with no file path and no verify line is not a step, it is a wish.
- Hunt the escape hatches before they go in. `as any`, `!`, `// @ts-ignore`, implicit returns, missing annotations. If the plan needs one of these to work, that is a Decision, not a hidden assumption.
- No compliments, no hedging. If you cannot make the call, that is a Decision row, not weasel words. (The exact banned phrases live in the pre-output gate so they only have to be maintained in one place.)
- Don't manufacture risks or out-of-scope items to look thorough. If the section is empty, omit it.

## When intel is present

Intel is an upstream dependency, not flavor. Every intel section maps to a named place in the plan:

- **Story Context** → header line `**Story (from intel):** <AB-id> — <one-line scope> (parent: <AB-parent-id> — <parent title>)`. Omit the entire line when no intel was found in Pre-Plan step 2.
- **Scope (verbatim ADO description + build targets)** → seeds section `1. What you're doing`. Quote the relevant fragment; don't paraphrase.
- **AC-N / Findings** → carry into `0. Verified context` as bullets. Trust intel's cite (`Source: <path:line>`); don't re-grep. Annotate the bullet with the intel AC label if applicable (e.g. `(AC-2)`).
- **Open Questions** → mandatory placement. Each open question must end up in **one** of:
  - Answered in the plan, with its own cite to the answer.
  - A Decision row in section 3 (the question is a real fork).
  - Section `4. Not in scope`, annotated `(intel open Q)`.
  - No silent drops. If you finish the plan with an unhandled open question, you're not done.

When intel is **stale** (`Last gathered:` > 14 days from today), add a header line `**Intel:** stale (gathered N days ago, consider /rick-intel <AB-id> --refresh)` and print a warning at Confirm and stop.

When intel **does not exist** (silent or gated-suggestion fallthrough), omit every intel-derived line above and proceed normally.

## Output structure

See [OUTPUT.md](OUTPUT.md) for the full section layout (Header, 0. Verified context, 1. What you're doing, 2. The plan, 3. Decisions, 4. Not in scope) with worked examples. Read it before drafting the plan body.

## Before you write the plan

Four checks. Run them against the draft in your head before any file gets written. If any check fails, fix the draft, do not write yet.

- Every step has `Files:` with a real path (not "the auth module") and `Verify:` with a real test name, `tsc --noEmit`, or a manual check. If a step is missing either, name the files, add the verify, or delete the step. (Splitting is a remedy for size, not for vagueness — the size rule is in [OUTPUT.md](OUTPUT.md) under section 2.)
- Every Verified context bullet has a citation `(path:N)` or `(path:symbol)`. Intel-carried bullets keep intel's `Source: <path:line>` cite verbatim. Either form satisfies; bullets with no citation get removed.
- Every Decision row names what gets worse in the Tradeoff line. If you cannot name what gets worse, the fork is not real — drop the row.
- Every intel Open Question is accounted for (answered, Decision row, or Not in scope per "When intel is present"). If one is unhandled, you're not done.
- No "we could," "one option is," "consider," "might want to," or "alternatively" anywhere in the body. If you find one, delete the sentence and either commit or move to Decisions.

## Write the plan

Before writing anything: confirm `docs/rick/<folder>/plan/current.md` does NOT already exist (or Morty explicitly said "rewrite from scratch"). If it exists and no rewrite was requested, you are in the wrong section — go back to "If a plan already exists" and route to `/rick-plan-improve`. Writing over an existing canonical destroys Morty's prior plan.

Two flows. Pick one before touching the filesystem.

**Fresh write** (no canonical exists):

1. Write the canonical at `docs/rick/<folder>/plan/current.md`. Use the section structure from [OUTPUT.md](OUTPUT.md). Cite paths and line numbers everywhere.
2. Copy the canonical to `docs/rick/<folder>/plan/v1.md` as the initial snapshot.
3. Create `docs/rick/<folder>/plan/VERSIONS.md` with this content:

   ```markdown
   # Plan Versions: <folder>

   Append-only history. v1 is the initial plan; later versions are written by /rick-plan-improve.

   - v1 | <YYYY-MM-DD> | initial plan
   ```

**Rewrite from scratch** (canonical exists AND Morty explicitly asked for a full rewrite — rare, prefer `/rick-plan-improve`): do NOT run the fresh-write steps, step 2 would clobber the original `v1.md` snapshot. Instead:

1. Let `<N>` = (highest existing `v<int>` entry in `<folder>/plan/VERSIONS.md`) + 1.
2. Copy the existing canonical to `<folder>/plan/v<N>-pre.md` (do NOT overwrite an older `v<int>-pre.md`).
3. Write the new plan to the canonical at `<folder>/plan/current.md`.
4. Snapshot the new canonical to `<folder>/plan/v<N>.md`.
5. Append exactly `- v<N> | <YYYY-MM-DD> | rewrite from scratch` to VERSIONS.md.

## Confirm and stop

Print exactly: `Planned "<folder>": docs/rick/<folder>/plan/current.md`

Warnings prepend the confirm line (in order, if multiple apply):

```
Warning: gh issue lookup failed for #<N>, used slug fallback
Warning: intel for <folder> is stale (N days), consider /rick-intel <AB-id> --refresh
Planned "<folder>": docs/rick/<folder>/plan/current.md
```

Then stop. **Do not start implementing.** The plan is the deliverable; Morty reads it and decides what runs. No summary, no "let me know if you want to adjust," no "shall I start on step 1," no first step pre-emptively executed. If you find yourself about to read a file or edit code after printing the path, you are violating the stop. Wait for the next message.
