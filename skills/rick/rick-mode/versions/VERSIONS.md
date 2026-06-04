# Versions: rick-mode

Append-only history of /rick-improve runs on this skill. Each entry is one bounded loop.

- v1 | 2026-06-04 | pre-loop snapshot
- v1 | 2026-06-04 | manual-improve (concrete user feedback, single round, did not run the bounded loop because findings were already pre-cited)
  - Round 1 (combined):
    - Added Agency rule. Reversible ops Rick runs himself with the tools; Morty only applies irreversible state changes. Default is execute, not lecture.
    - Added Narrate-while-doing rule. First-person present verbs while Rick is acting, not "you should" lectures.
    - Mode section split into explicit Review mode + Pairing mode branches. The "drop the section structure" one-liner was retrofitting pairing onto a review skill; Pairing mode now has its own protocol (execute, narrate, hand off only the irreversible bits).
    - Output structure renamed to "Review output structure" with scope note. Section 3 reframed from "What to do instead" (which biased everything to Morty-applies-the-fix) into a two-bucket split: "Doing now" (reversible work Rick runs in the same turn) and "Needs your call" (irreversible / judgment-loaded).
    - Stop condition split into Review-mode (sections done) and Pairing-mode (executable portion done or real hand-off point hit). Explicit anti-pattern: "If you just listed git commands and stopped, you stopped at the wrong place."
  - Round 2: skipped — manual targeted edit, not a discovery loop
  - Round 3: skipped — same
  - Restore pre when: someone needs the original review-only framing back (rare; the original conflated review and pairing)
  - Restore post when: this is the new baseline for any further /rick-improve runs
