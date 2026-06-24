# Versions: rick-improve

Append-only history of /rick-improve runs on this skill. Each entry is one bounded loop.

- v1 | 2026-06-12 | 3-rounds
  - Round 1: required structured finding format (category/where/failure-mode) with example, scoped reference bar to every round (not just round 1), reframed 3-round cap as a ceiling not a target, tightened zero-changes stop to exclude trivial polish
  - Round 2: added absolute-path source-repo as resolution priority 1 (installed copies clobber on resync), added stop condition 4 for whole-section rewrites, added concrete restore-condition examples
  - Round 3: cleaned up round-2 drift — added `needs-rewrite` token to stop-condition enum, clarified SC4 still snapshots post, disambiguated "same root" for reference bar after absolute-path resolution
  - Restore pre when: the structured-finding format makes rounds 2-3 too rigid for skills with broad design problems and we want freeform critique back
  - Restore post when: future /rick-improve runs on other skills stop manufacturing micro-polish in rounds 2-3 — confirms the cap-not-target framing and zero-changes teeth landed
