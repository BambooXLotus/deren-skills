# Versions: rick-review-comments

Append-only history of /rick-improve runs on this skill. Each entry is one bounded loop.

- v1 | 2026-06-12 | pre-loop snapshot
- v1 | 2026-06-12 | 3-rounds
  - Round 1: added `disable-model-invocation: true`; replaced P-tier schema-blob with rendered example; added concrete line format to "Order of operations"; added pre-write 3-check gate; gave the no-hedge voice rule teeth (strike + rewrite imperative).
  - Round 2: scoped the no-hedge rule to the ask only so the `why:` line isn't stripped; clarified BLOCKED is indistinguishable from no-marker in `current.md` and prescribed transcript-prepend behavior when `/rick-fix` chat is in context.
  - Round 3: tightened carry-over definition (absent-from-current.md only; strikethrough rows belong to "this round") to kill the duplicate-entry overlap with Step 3 item 4; dropped the "(optional)" framing that conflicted with the mandatory print line.
  - Restore pre when: a future rick-review change makes the source review file no longer have rick-fix markers (the routing table would mis-classify every row).
  - Restore post when: rick-review and rick-fix keep their current marker semantics — v1 is the baseline.
- v2 | 2026-06-12 | pre-loop snapshot
- v2 | 2026-06-12 | zero-changes
  - Round 1: added stop-discipline at end of Step 5 (no "want me to paste / commit / ping" pitbull closers); removed redundant voice bullets (`why:` and fenced-block presence) that the Step 3 pre-write gate already enforces; added a one-liner noting structural rules live in the gate, not the voice section.
  - Round 2: added rendered examples for Step 3 item 1 (header + context paragraph), item 3 (Not commenting kill row), and item 4 (Verified clean rows — disambiguated the slash suffix into two distinct sources with explicit "(this round)" vs "(carry-over from v<N-1>)" picks).
  - Round 3: stopped early — remaining candidates (Step 2.5 row 1 phrasing, Step 3 item 5 rendered example) are diminishing-return polish; failure modes low-frequency and mitigated by sibling spec sections. Forcing the round would be manufacturing pressure.
  - Restore pre when: a future rick-review revision changes the `current.md` header format (the Step 2 parse instructions would break before any v2 voice/example refinement matters).
  - Restore post when: rick-review's `current.md` header keeps the `**Version:** v<N>` + branch + base + scope + verdict shape — v2 is the baseline.
