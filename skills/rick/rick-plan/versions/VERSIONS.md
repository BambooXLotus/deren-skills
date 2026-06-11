# Versions: rick-plan

Append-only history of /rick-improve runs on this skill. Each entry is one bounded loop.

- v1 | 2026-06-11 | pre-loop snapshot
- v1 | 2026-06-11 | 3-rounds
  - Round 1: added pre-output gate ("Before you write the plan"), concrete example for the step format, concrete example for the Decisions row, and teeth on "Pick. Don't enumerate." (consequence + banned phrases).
  - Round 2: promoted the "existing plan = revise, don't overwrite" rule out of Pre-Plan step 5 into its own "If a plan already exists" block, and merged the dangling "Stop condition" section into "Confirm and stop" so the load-bearing "Do not start implementing" rule isn't demoted to a footer.
  - Round 3: added an explicit existence-check precondition at the top of "Write the plan" so a model jumping straight from the protocol to the write step can't silently overwrite the canonical.
  - Restore pre when: the example steps or example Decisions row start getting copied verbatim into real plans (token-rotation specifics leaking into unrelated features) — the examples are too sticky and the templates without examples are safer.
  - Restore post when: the pre-output gate visibly prevents at least one bad plan from shipping (uncited Verified context, vague Files, manufactured decision), or the existence check at "Write the plan" catches at least one would-be overwrite.
- v2 | 2026-06-11 | pre-loop snapshot
- v2 | 2026-06-11 | zero-changes
  - Round 1: trimmed redundant hedging-phrase list out of the Rules section (line 77 now points at the pre-output gate as the single source of truth), simplified the inline "no 'we could try'" ban inside Section 1's spec, and disambiguated the step-split rule ("more than one test file" rather than "more than one test", with a clarifying parenthetical that multiple `should` cases in one file is fine).
  - Round 2: removed the explicit banned-phrase list from line 73 too, leaving line 149 (the pre-output gate) as the only place the phrases are enumerated. Resolves the contradiction with line 77 introduced in round 1.
  - Round 3: stopped early. No improvements found this round — remaining candidates (rick-mode-coupling note on line 86, soft 500-word cap, header three-line vs comma-list ambiguity) failed the "name the failure mode" test and would have been manufactured findings.
  - Restore pre when: future edits to the banlist at line 149 fail to propagate and the rules silently drift from the gate (the "single source of truth" coupling assumes future-me actually keeps updating the gate, not the rules). If that fails, v1's redundant lists at least kept the warnings visible at the Rules level even when the gate was out of sync.
  - Restore post when: any future /rick-improve run wants to extend the banlist by editing one place — the gate at line 149 — and have the Rules sections inherit by reference. The v2 structure pays off the moment that happens.
- v3 | 2026-06-11 | pre-loop snapshot
- v3 | 2026-06-11 | zero-changes
  - Round 1: replaced the "split it" remedy in the pre-output gate (line 146) with "name the files, add the verify, or delete the step" since splitting a vague step yields two vague steps; also added an ordering note to Pre-Plan Protocol step 5 (line 23) flagging that `<folder>` must be resolved via "Pick the folder" first because step 5 reads `docs/rick/<folder>/plan/current.md`.
  - Round 2: tightened the rewrite-from-scratch branch under "Write the plan" (line 167) — defined `<N>` as "highest existing v<int> in VERSIONS.md, plus one," forbade overwriting older `v<int>-pre.md` snapshots, and specified the exact appended VERSIONS.md line text (`- v<N> | <YYYY-MM-DD> | rewrite from scratch`) so the rare full-rewrite path can't clobber history accumulated by /rick-plan-improve.
  - Round 3: stopped early. Remaining candidates (header three-line format example, rick-mode coupling on line 86, branch banlist completeness, slugify 40-char edge case with no hyphens) all failed the "name a specific failure mode" test and would have been manufactured findings.
  - Restore pre when: the new ordering note in step 5 makes a reader assume the whole protocol is reorder-on-demand and they start rearranging other steps that genuinely must run in order. If that happens, v2's simpler step 5 was less prescriptive.
  - Restore post when: a real rewrite-from-scratch invocation comes through and the version-increment math (highest + 1, never overwrite older pre snapshot) saves you from clobbering /rick-plan-improve history, OR a model writes a vague step that the gate would have papered over with "split it" before round 1.
- v4 | 2026-06-11 | pre-loop snapshot
- v4 | 2026-06-11 | zero-changes
  - Round 1: no improvements found this round — strongest candidates (header three-line vs comma-list ambiguity, $ARGUMENTS handling when on a feature branch with a leading word that looks like a slug/issue#, "Test pattern" bullet lacking line citation, banlist gap for step 1 branch names) all fail the "name a specific recurring failure mode" test, and the header candidate would directly override v3's explicit rejection of the same finding written on the same day with no time for drift to justify revisiting.
  - Round 2: stopped early
  - Round 3: stopped early
  - Restore pre when: not applicable — v4-pre.md and v4.md are byte-identical
  - Restore post when: not applicable — no change to restore
- v5 | 2026-06-11 | pre-loop snapshot
- v5 | 2026-06-11 | zero-changes
  - Round 1: tightened Pre-Plan Protocol step 5 (line 23) to short-circuit on existing canonical — model now jumps to "If a plan already exists" before reading the old plan as input, blocking the "read it then write an improved version over the canonical" failure mode. Split "Write the plan" (line 151) into explicit **Fresh write** and **Rewrite from scratch** labeled flows, with a "do NOT run the fresh-write steps, step 2 would clobber the original `v1.md` snapshot" guard on the rewrite branch to prevent silent snapshot destruction.
  - Round 2: updated the cross-reference at line 68 from "the version-increment flow under 'Write the plan'" to "the **Rewrite from scratch** flow under 'Write the plan'" — direct cleanup of the stale label round 1's split created.
  - Round 3: stopped early. Remaining candidates (line 146 hard line-number reference "see line 116 for the size rule" — latent fragility but currently accurate; line 90 "Test pattern" and line 91 "CLAUDE.md:" example bullets without `(path:N)`; line 84 header three-line format) all either fail the "name a present failure mode" test or were explicitly rejected by v2/v3/v4 with valid reasoning. No drift since v4 to justify revisiting.
  - Restore pre when: a model in rewrite-from-scratch mode is observed reading the existing plan as "context" then writing a fundamentally disconnected new plan — round 1's step 5 short-circuit might be over-aggressive in cutting off legitimate context-gathering for the rare rewrite path. v4-pre's looser step 5 wording was less prescriptive.
  - Restore post when: the round 1 step 5 short-circuit catches at least one would-be "read-and-overwrite" run that the v4 routing rule would have caught too late, OR the round 1 "Rewrite from scratch" labeled flow prevents a real v1.md snapshot clobbering in a rewrite-mode invocation.
