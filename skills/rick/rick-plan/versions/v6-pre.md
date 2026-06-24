# Pick the folder (rick-plan)

The folder ties this plan to future saves, recaps, respawns, and reviews under the same identity.

Unlike [`../rick-review/RESOLVE-FOLDER.md`](../rick-review/RESOLVE-FOLDER.md) — which identifies an *existing* feature thread — this picker *creates* one. It handles the off-branch case where a bare issue number or a slug is the seed.

## Resolution order (use the first that produces a value)

1. **Branch name.** `git branch --show-current`. If non-empty and not in `{main, master, develop}`, sanitize by replacing `/` and spaces with `-`. That is the folder. Example: `48-rotate-share-token` stays as-is; `feature/auth-cleanup` becomes `feature-auth-cleanup`.

2. **Issue-number argument.** Only fires when step 1 did NOT resolve. If the first whitespace-delimited word of `$ARGUMENTS` matches `^[0-9]+$` AND `gh` is on PATH (`command -v gh >/dev/null`) AND the issue exists, run:

   ```bash
   gh issue view <N> --json number,title --jq '"\(.number)-\(.title)"'
   ```

   Slugify the returned `<number>-<title>` as follows, in order:
   1. If the string contains an em-dash (`—`), en-dash (`–`), or space-hyphen-space (` - `), truncate at the FIRST such delimiter (drop the delimiter and everything after it). Issue titles overwhelmingly use these as "main clause — qualifier" boundaries; the qualifier is fluff for folder-naming purposes.
   2. Lowercase the whole string.
   3. Replace every run of characters NOT in `[a-z0-9]` with a single hyphen.
   4. Trim leading and trailing hyphens.
   5. If the result exceeds 40 characters total, truncate to the last hyphen position that keeps the result ≤ 40 chars (never cut mid-segment); strip any trailing hyphen.

   The result is the folder. Example: issue #50 "Rotate share link UI — button + confirmAndRun on event detail" → `50-rotate-share-link-ui`.

   Strip the leading number from `$ARGUMENTS` before treating the rest as the goal.

   If `gh issue view` fails (no `gh` on PATH, not authenticated, wrong repo, issue deleted, network down), fall through to step 3 and surface a one-line warning in the final printed output so Morty knows the issue lookup missed: `Warning: gh issue lookup failed for #<N>, used slug fallback`.

3. **Slug override.** If on main/develop/detached HEAD and the first whitespace-delimited word of `$ARGUMENTS` matches `^[a-z0-9][a-z0-9-]*$` and is not in the banlist, use it as the folder. Strip it from `$ARGUMENTS` before treating the rest as the goal. A bare digit string (`50`) only reaches this step when step 2 fell through; it is used verbatim as the folder name (with the warning), which is suboptimal but better than silently inventing a slug.

4. **Derived slug.** Pick a one-word slug from the request. Prefer the domain (`auth`, `billing`, `onboarding`, `tooling`, `migration`) over the activity (`refactor`, `fix`, `add`). Max 20 characters, lowercase, hyphens allowed (`user-import`), no underscores.

## Banlist

Steps 3 and 4 only — never applies to branch names or issue-derived folders.

Banned: `stuff`, `session`, `work`, `code`, `dev`, `misc`, `general`, `task`, `thing`, `update`, `changes`, `wip`, `rick`, `plan`.

If you almost picked one, the slug is too vague. Read more context and pick again.

## Don't dedupe against siblings

Never use a folder derived from a prior sibling's slug. If `docs/rick/` already contains other folders for unrelated features, that has zero bearing on this resolution — folders are not deduplicated, reused, or "matched" against prior siblings.
