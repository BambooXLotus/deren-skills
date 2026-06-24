# Intel lookup (for other rick-* skills)

How to find a rick-intel dossier before going out and re-sweeping vexis-docs / ADRs / ADO yourself. Used by `rick-plan`, `rick-mode`, `rick-plan-improve`, `rick-review`, and any skill that gathers domain context.

The dossier sits at `docs/rick/<folder>/intel/current.md`, where `<folder>` matches the branch name (intel writes there directly when run on a feature branch, or to the predicted branch name `<digits>-<title-slug>` when run pre-branch). So the most common lookup is just same-folder.

## Find the dossier (first that hits wins)

1. **Same folder.** `docs/rick/<current-folder>/intel/current.md` where `<current-folder>` is whatever the caller already resolved (sanitized branch name, plan folder, etc.). This is the common case once a branch exists — intel and everything else co-locate.
2. **AB-id glob.** If the caller knows an AB-id (from `$ARGUMENTS`, branch name, or PR body), scan for any prior intel that mentions it:
   ```bash
   find docs/rick -type f -path '*/intel/current.md' -exec grep -l '<AB-id>' {} \;
   ```
   Single hit → use it. Multiple hits → pick the newest by `Last gathered:` header; note the others under "Open threads" so Morty can clean up.
3. **No intel.** Fall through to whatever the caller would have done without it.

## Resolving the AB-id (when the caller doesn't already have one)

Priority, first that produces a value:

1. **Explicit arg.** `$ARGUMENTS` first token matches `^(AB|ab)?\d{2,}$` (e.g. `AB16289`, `ab16289`, `16289`) → normalize to `AB<digits>`.
2. **Branch name.** `git branch --show-current` and grep for `(AB|ab)\d{2,}` anywhere in it (e.g. `feat/ab16289-foo`, `16289-rotate`).
3. **PR body.** `gh pr view --json body 2>/dev/null` and grep the body for `(AB|ab)\d{2,}`.

## What to do once you've got the path

Read the file. Use these sections as inputs (the file's section order is in [OUTPUT.md](OUTPUT.md)):

- **Story Context** — parent, state, iteration, PR, depends, blocks.
- **Scope** — verbatim ADO description + build targets.
- **AC-N / Findings** — every cited fact already gathered. Each line has `Source: <path:line>`; trust the cite, don't re-grep.
- **Open questions** — what the dossier couldn't answer. Flag these forward in the caller's output.

## What this saves

The intel sweep already grepped `vexis-docs/` and `docs/adr/` for the story's keywords, and read the matching domain anchor in full. If intel exists, the caller doesn't need to repeat any of that. Source code is **not** covered by intel — the caller still reads the actual codebase for the work it's planning / reviewing / fixing.

## Staleness

Each finding carries a `Found: <YYYY-MM-DD>`. The dossier's `Last gathered:` header is the most recent gather. If the date is more than 14 days old, the caller should note `intel stale (gathered N days ago)` in its output and may want to suggest `/rick-intel <AB-id> --refresh`.
