# Intel lookup (for other rick-* skills)

How to find a rick-intel dossier before going out and re-sweeping vexis-docs / ADRs / ADO yourself. Used by `rick-plan`, `rick-mode`, and any other skill that gathers domain context.

The dossier sits at `docs/rick/<AB-id>/intel/current.md` (e.g. `docs/rick/AB16289/intel/current.md`). The folder is the ADO id, not the branch name — so the lookup has to resolve the AB-id first.

## Resolve the AB-id (first that produces a value wins)

1. **Explicit arg.** If the caller's `$ARGUMENTS` first token matches `^(AB|ab)?\d{2,}$` (e.g. `AB16289`, `ab16289`, `16289`), normalize to `AB<digits>` and use it.
2. **Branch name.** `git branch --show-current` and grep for `(AB|ab)\d{2,}` anywhere in it (e.g. `feat/ab16289-foo`, `16289-rotate`). Normalize to `AB<digits>`.
3. **PR body.** `gh pr view --json body 2>/dev/null` and grep the body for `(AB|ab)\d{2,}`. Normalize.
4. **Same-folder fallback.** Whatever folder the caller already resolved (branch-folder, slug, etc.). Check `docs/rick/<that-folder>/intel/current.md` directly — supports the case where intel was co-located under the feature folder.

If nothing resolves, no intel exists for this work. Fall through to whatever the caller would have done without intel.

## Check the path

```bash
test -f docs/rick/<AB-id>/intel/current.md
```

If present, read the file. Use these sections as inputs (the file's section order is in [OUTPUT.md](OUTPUT.md)):

- **Story Context** — parent, state, iteration, PR, depends, blocks.
- **Scope** — verbatim ADO description + build targets.
- **AC-N / Findings** — every cited fact already gathered. Each line has `Source: <path:line>`; trust the cite, don't re-grep.
- **Open questions** — what the dossier couldn't answer. Flag these forward in the caller's output.

## What this saves

The intel sweep already grepped `vexis-docs/` and `docs/adr/` for the story's keywords, and read the matching domain anchor in full. If intel exists, the caller doesn't need to repeat any of that. Source code is **not** covered by intel — the caller still reads the actual codebase for the work it's planning / reviewing / fixing.

## Staleness

Each finding carries a `Found: <YYYY-MM-DD>`. The dossier's `Last gathered:` header is the most recent gather. If the date is more than 14 days old, the caller should note `intel stale (gathered N days ago)` in its output and may want to suggest `/rick-intel <AB-id> --refresh`.
