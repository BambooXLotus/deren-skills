---
name: rick-improve
description: Self-improvement loop for any rick-* skill prompt. Bounded 3-round critique-and-fix loop targeting skills/rick/<name>/SKILL.md (or installed equivalent). Snapshots pre/post versions to a versions/ directory next to the target. 250-line cap. Use when user invokes /rick-improve <skill-name>, says "improve rick-plan", "tighten the prompt", or wants to refine a skill prompt without rewriting from scratch.
argument-hint: <skill-name> (e.g. rick-plan, rick-mode)
disable-model-invocation: true
---

# Rick Improve

Bounded self-improvement loop on a Rick skill prompt. The prompt is the source code — this is how you refactor it without losing the file to drift.

Target skill: `$ARGUMENTS`

If `$ARGUMENTS` is empty, tell Morty to give a target skill name and stop. Don't guess.

## Resolve the target

Look in priority order, use the first match:

1. `skills/rick/$ARGUMENTS/SKILL.md` (source repo)
2. `.claude/skills/$ARGUMENTS/SKILL.md` (installed location)

If neither exists, list available rick skill directories from whichever root resolved and stop. Don't try to be clever.

Below, `<target-dir>` is the directory holding the resolved SKILL.md.

## Reference bar

Read `rick-mode/SKILL.md` from the same root (source or install). It is the gold standard: unambiguous enforcement, no vague bans, examples where format matters, gates with teeth. Use it as the bar for round-1 critique. Anything the target does worse than rick-mode is a candidate finding.

## Before the loop

1. Read the target SKILL.md. The actual file.
2. Determine version number: list `<target-dir>/versions/v*.md`, find the highest integer (ignore `-pre` suffix), next = N+1. If the directory or files don't exist, this is v1.
3. Snapshot: copy current SKILL.md → `<target-dir>/versions/v<N>-pre.md`. Create the `versions/` dir if needed.
4. Append a stub line to `<target-dir>/versions/VERSIONS.md` (create with the header below if missing): `- v<N> | <YYYY-MM-DD> | pre-loop snapshot`
5. Count body lines (excluding frontmatter). If already over 250, the first round must trim before it can add anything.

## Stop conditions (first one that fires, stop immediately)

1. **3 rounds completed.** Hard cap.
2. **Round produces zero net-new changes.** Say "no improvements found this round" and stop.
3. **Body exceeds 250 lines.** Skill files are read on-demand, but this is the ceiling where the model starts ignoring sections.

## Loop protocol (repeat up to 3 times)

Each round:

1. **Read the current file.** The actual file, not your memory of it.
2. **Critique against the weakness categories below.** Cite the line or section that's broken.
3. **Rank problems worst first.** For each one, name the specific failure mode it causes in one sentence. If you can't name the failure mode, the problem does not qualify. Skip it.
4. **Apply surgical fixes.** Don't rewrite sections that work.
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

If creating for the first time, start the file with:

```markdown
# Versions: <skill-name>

Append-only history of /rick-improve runs on this skill. Each entry is one bounded loop.
```

**Pre-loop stub (written before round 1):**

```
- v<N> | <YYYY-MM-DD> | pre-loop snapshot
```

**Post-loop full entry (appended after the loop finishes):**

```
- v<N> | <YYYY-MM-DD> | <stop-condition: 3-rounds | zero-changes | over-cap>
  - Round 1: <one line, what changed>
  - Round 2: <one line, or "stopped early">
  - Round 3: <one line, or "stopped early">
  - Restore pre when: <condition that would make v<N>-pre better than v<N>>
  - Restore post when: <condition that would make v<N> the new baseline>
```

## Confirm and stop

After the post snapshot is written and VERSIONS.md is updated, print exactly:

```
Improved <skill-name> (v<N>): <stop-condition>. History: <target-dir>/versions/VERSIONS.md
```

Then stop. The round-by-round summary is already in VERSIONS.md. No recap. No "want me to revert."
