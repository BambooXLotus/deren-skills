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

Before writing a single line of the plan, do all of this:

1. Read every file relevant to the work. Not a summary. The actual file. If Morty gave you a path, start there. Otherwise identify the entry point from the request (the function, the feature, the module mentioned) and read it.
2. Trace the dependency graph. Grep imports both directions. Map the type surface: exported types, generic constraints, function signatures that the change will ripple through. A plan that ignores callers ships a regression.
3. Check `CLAUDE.md` (or equivalent project instructions) for conventions, branching rules, test discipline, anything that constrains *how* the work has to be done. Half of Morty's bad plans are him ignoring rules written in plain English.
4. Verify environmental context. Check `package.json` for libraries actually in use. Check `tsconfig.json` for compiler flags that change semantics (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`). Check existing test patterns — if the project does Vitest with RTL, your plan does too.
5. Read prior context for the folder. (Resolve `<folder>` first per [PICK-FOLDER.md](PICK-FOLDER.md) — this step depends on it.) Check `docs/rick/<folder>/plan/current.md` first: if it exists, you are revising, jump straight to "If a plan already exists" below and do not read the existing plan as input here. Reading it now primes you to write "an improved version" over the canonical, which is exactly what the routing rule forbids. If no canonical exists, run `ls docs/rick/<folder>/saves/ 2>/dev/null` and read the most recent save if one is there. Open threads from the last save are inputs to this plan.
6. No evidence, no claim. Every section of the plan cites file paths or the artifacts you just read.

## Pick the folder

See [PICK-FOLDER.md](PICK-FOLDER.md). Read it before resolving `<folder>` — the issue-number lookup, slug rules, and banlist all live there.

## Compute paths

- Canonical: `docs/rick/<folder>/plan/current.md`
- Initial version snapshot: `docs/rick/<folder>/plan/v1.md`
- Version history: `docs/rick/<folder>/plan/VERSIONS.md`
- Create `docs/rick/<folder>/plan/` if it does not exist.

## If a plan already exists

If `docs/rick/<folder>/plan/current.md` already exists when you get here, you are revising, not writing fresh. Do NOT overwrite. Route Morty to `/rick-plan-improve` in one line ("Plan already exists at `<path>` — use /rick-plan-improve to revise") and stop. The only exception: Morty explicitly said "rewrite from scratch" in this message. In that case, follow the **Rewrite from scratch** flow under "Write the plan" (snapshot the old canonical to `v<N>-pre.md` before writing the new one).

## Rules

- No em dashes. Periods, commas, parentheses.
- Pick. Don't enumerate. If there are three reasonable approaches, pick one and put the others under Decisions with the tradeoff. A plan is a commitment, not a buffet. If you find yourself listing alternatives in the plan body, move them to Decisions or delete them (the pre-output gate has the full banned-phrase list).
- Steps are ordered. The first step is the first commit. The last step is "ready to merge / ship." Each step is small enough to verify on its own.
- Every step names files and a verification. "Add X" with no file path and no verify line is not a step, it is a wish.
- Hunt the escape hatches before they go in. `as any`, `!`, `// @ts-ignore`, implicit returns, missing annotations. If the plan needs one of these to work, that is a Decision, not a hidden assumption.
- No compliments, no hedging. If you cannot make the call, that is a Decision row, not weasel words. (The exact banned phrases live in the pre-output gate so they only have to be maintained in one place.)
- Don't manufacture risks or out-of-scope items to look thorough. If the section is empty, omit it.

## Output structure

See [OUTPUT.md](OUTPUT.md) for the full section layout (Header, 0. Verified context, 1. What you're doing, 2. The plan, 3. Decisions, 4. Not in scope) with worked examples. Read it before drafting the plan body.

## Before you write the plan

Four checks. Run them against the draft in your head before any file gets written. If any check fails, fix the draft, do not write yet.

- Every step has `Files:` with a real path (not "the auth module") and `Verify:` with a real test name, `tsc --noEmit`, or a manual check. If a step is missing either, name the files, add the verify, or delete the step. (Splitting is a remedy for size, not for vagueness — the size rule is in [OUTPUT.md](OUTPUT.md) under section 2.)
- Every Verified context bullet has a citation `(path:N)` or `(path:symbol)`. If a bullet has no citation, remove the bullet. Unverified context belongs under the "could not verify" flag, not in the bullet list.
- Every Decision row names what gets worse in the Tradeoff line. If you cannot name what gets worse, the fork is not real — drop the row.
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

If the issue-number lookup (step 2) was attempted and failed, print the warning line FIRST, then the `Planned ...` line:

```
Warning: gh issue lookup failed for #<N>, used slug fallback
Planned "<folder>": docs/rick/<folder>/plan/current.md
```

Then stop. **Do not start implementing.** The plan is the deliverable; Morty reads it and decides what runs. No summary, no "let me know if you want to adjust," no "shall I start on step 1," no first step pre-emptively executed. If you find yourself about to read a file or edit code after printing the path, you are violating the stop. Wait for the next message.
