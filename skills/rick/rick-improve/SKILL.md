---
name: rick-improve
description: Bounded 3-round critique-and-fix loop on a rick-* skill prompt. Snapshots pre/post versions next to the target. 250-line cap.
argument-hint: <skill-name> (e.g. rick-plan, rick-mode)
disable-model-invocation: true
---

# Rick Improve

Bounded self-improvement loop on a Rick skill prompt. The prompt is the source code — this is how you refactor it without losing the file to drift.

Target skill: `$ARGUMENTS`

If `$ARGUMENTS` is empty, tell Morty to give a target skill name and stop. Don't guess.

## Resolve the target

Look in priority order, use the first match:

1. An absolute path to the skills source repo if Morty's memory, CLAUDE.md, or this conversation references one (e.g. `~/code/deren-skills/skills/rick/$ARGUMENTS/SKILL.md`). Installed copies get clobbered on resync — always prefer the source of truth when known.
2. `skills/rick/$ARGUMENTS/SKILL.md` (source repo, relative to cwd)
3. `.claude/skills/$ARGUMENTS/SKILL.md` (installed location, last resort)

If none exists, list available rick skill directories from whichever root resolved and stop. Don't try to be clever.

Below, `<target-dir>` is the directory holding the resolved SKILL.md.

## Reference bar

Read `rick-mode/SKILL.md` from the same root as the resolved target (i.e. the `rick-mode/` sibling of `<target-dir>`). It is the gold standard: unambiguous enforcement, no vague bans, examples where format matters, gates with teeth. Use it as the bar for **every round's** critique. Anything the target does worse than rick-mode is a candidate finding.

## Before the loop

1. Read the target SKILL.md. The actual file.
2. Determine version number: list `<target-dir>/versions/v*.md`, find the highest integer (ignore `-pre` suffix), next = N+1. If the directory or files don't exist, this is v1.
3. Snapshot: copy current SKILL.md → `<target-dir>/versions/v<N>-pre.md`. Create the `versions/` dir if needed.
4. Append a stub line to `<target-dir>/versions/VERSIONS.md` (create with the header below if missing): `- v<N> | <YYYY-MM-DD> | pre-loop snapshot`
5. Count body lines (excluding frontmatter). If already over 250, the first round must trim before it can add anything.

## Stop conditions (first one that fires, stop immediately)

1. **3 rounds completed.** Hard cap.
2. **Round produces zero net-new changes.** Trivial polish (whitespace, word swaps without semantic change) does not count. Say "no improvements found this round" and stop.
3. **Body exceeds 250 lines.** Skill files are read on-demand, but this is the ceiling where the model starts ignoring sections.
4. **A finding needs a whole-section rewrite.** That's a different skill (`/rick-plan-improve`), not this one. Still snapshot post (so the partial round's edits are captured) and tag the entry `needs-rewrite`.

## Loop protocol (repeat up to 3 times)

3 is a cap, not a target. Most skills converge in 1-2 rounds. If round 2 or 3 starts and you cannot name a real failure mode, stop early — don't dig for micro-polish to "use the budget."

Each round:

1. **Read the current file.** The actual file, not your memory of it.
2. **Critique against the weakness categories below.** Every finding uses this format:

   **[weakness category from list below]**
   - **Where:** `SKILL.md:<line>` or quoted snippet
   - **Failure mode:** one sentence naming what the model does wrong because of this

   Example:

   **Vague enforcement**
   - **Where:** SKILL.md:42 — "be specific in your critique"
   - **Failure mode:** model writes "this section is unclear" with no line citation, leaving the next round nothing concrete to act on.

3. **Rank problems worst first.** If you can't fill the **Failure mode** line, the problem does not qualify. Skip it.
4. **Apply surgical fixes.** Edit specific lines, not whole sections. (Whole-section rewrite triggers Stop condition 4.)
5. **Count body lines after edits.** If over 250, stop the loop immediately regardless of round number.
6. **Report what changed and why.** One line per change. Hold these for the post-loop VERSIONS.md entry.

## After the loop

1. Snapshot: copy final SKILL.md → `<target-dir>/versions/v<N>.md` (post-loop result, bare integer matches the convention used by `rick-plan-improve` and `rick-review`).
2. Append the full entry to `<target-dir>/versions/VERSIONS.md` using the format below.

## Weakness categories

- **Redundancy:** two sections saying the same thing. The model picks one and ignores the other.
- **Missing examples:** a format requirement with no example. The model freestyles it wrong.
- **Unconditional rules that should be conditional:** "always do X" when X should only fire in specific situations.
- **Buried critical instructions:** anything that must survive context compression needs its own labeled block, not buried in a paragraph.
- **Vague enforcement:** "be specific" is not enforceable. "cite file:line or paste the snippet" is.
- **Overlapping sections:** if two sections could swap content without anyone noticing, differentiate them or merge.
- **Rules with no teeth:** a rule that says what to do but not what happens if you don't. Add the consequence.
- **Manufacturing pressure:** phrasing that implies "keep looking until you find something." Creates pressure to invent findings.

## What NOT to do

- Don't add sections because they seem useful. Every section must prevent a specific failure mode you can name.
- Don't expand examples into tutorials. One tight example per format requirement.
- Don't touch `versions/` files mid-loop. Only write pre (before round 1) and post (after the final round).
- Don't change the skill's frontmatter `name` field. Renaming the skill breaks every downstream reference.

## VERSIONS.md format

See [VERSIONS-FORMAT.md](VERSIONS-FORMAT.md) for the file header, pre-loop stub shape, post-loop full entry shape, and restore-condition rules. Read it before the first write (the pre-loop stub in step 4 of "Before the loop").

## Confirm and stop

After the post snapshot is written and VERSIONS.md is updated, print exactly:

```
Improved <skill-name> (v<N>): <stop-condition>. History: <target-dir>/versions/VERSIONS.md
```

Then stop. The round-by-round summary is already in VERSIONS.md. No recap. No "want me to revert."
