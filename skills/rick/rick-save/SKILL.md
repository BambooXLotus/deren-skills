---
name: rick-save
description: Save the current session's warm thread to docs/rick/<folder>/saves/<timestamp>.md so a future Claude can resume via /rick-respawn. Use when the user wants to hand off this thread to the next session.
argument-hint: "[optional first word: folder override (only used when not on a feature branch). Rest of arguments: extra focus for the next session]"
---

# Rick Save

Session save point. Captures the current warm thread to `docs/rick/<folder>/saves/<timestamp>.md` so a future Claude session can pick up exactly where this one left off via `/rick-respawn`. `<folder>` is the current branch name (e.g. `48-rotate-share-token`), or a one-word slug fallback when not on a feature branch.

Saves are append-only. Each session adds a new timestamped file under the feature's `saves/` directory. Reviewing the directory shows the full timeline for that feature.

Tighter focus than a generic handoff: the next agent gets what to do *right now*, not a narrative summary of the whole session.

## Protocol

### 1. Gather context (parallel, mandatory)

Run all of these. Read actual output, do not assume.

- `git status`
- `git branch --show-current`
- `git log -5 --oneline`
- `git diff --stat $(git merge-base HEAD main 2>/dev/null)..HEAD 2>/dev/null || git diff --stat HEAD~5..HEAD 2>/dev/null`
- Check `docs/rick/<folder>/saves/` exists. If not, create it. If `docs/rick/README.md` is missing, copy [README-TEMPLATE.md](README-TEMPLATE.md) to it verbatim.

If git commands fail, note under `What NOT to do` (or skip the affected sections and flag the gap).

### 2. Pick the folder

The folder ties this save to the plan, review, and recap for the same feature.

Resolution order, use the first that produces a value:

1. **Branch name.** `git branch --show-current`. If non-empty and not in `{main, master, develop}`, sanitize `/` and spaces to `-`. That is the folder. Example: `48-rotate-share-token` stays as-is.
2. **Slug override.** If on main/develop/detached HEAD and the first whitespace-delimited word of `$ARGUMENTS` matches `^[a-z0-9][a-z0-9-]*$` and is not in the banlist, use it as the folder. Strip it from `$ARGUMENTS` before using the rest as focus.
3. **Derived slug.** Pick a one-word slug from session context. Prefer the domain (`auth`, `billing`, `onboarding`, `tooling`, `migration`) over the activity (`review`, `fix`, `debug`).

Slug rules (paths 2 and 3 only): lowercase, alphanumeric plus hyphen, no spaces, no underscores. Single word strongly preferred. Hyphenated allowed when one word genuinely cannot disambiguate (`user-import`, `legacy-migration`). Max 20 characters.

Banlist (slug paths only): `stuff`, `session`, `work`, `code`, `dev`, `misc`, `general`, `task`, `thing`, `update`, `changes`, `wip`, `rick`, `save`. If you almost picked one, the slug is too vague. Pick again.

### 3. Compute filename

- Timestamp: `date +%Y-%m-%d_%H%M`
- Path: `docs/rick/<folder>/saves/<timestamp>.md`

### 4. Write the save

Second person, addressed to the next Claude. Direct and dense. No fluff, no pleasantries. 400 word soft cap.

Sections in this order:

- **Read first.** One or two sentences. Point at any prior save files in `docs/rick/<folder>/saves/` (recent timestamps) and the canonical plan at `docs/rick/<folder>/plan/current.md` that cover earlier context. Cite their paths. Do not duplicate their content.
- **Current state.** Branch, HEAD commit (short SHA), working tree status (clean / staged / dirty paths), PR number and state if applicable. Cite exact SHAs.
- **What just happened.** Bullet list. Deltas since the last referenced save (or this session, if this is the first save for the folder). Cite commit hashes for anything that landed.
- **Open threads.** Priority-ordered. Top one is the explicit next move. Each item: what it is, what blocks it, where it lives (file path with line numbers, PR number, branch). The next agent picks the top one and runs.
- **Lessons surfaced.** Non-obvious things learned this segment that aren't already in `CLAUDE.md` / `CLAUDE.local.md`. Skip the section if there are none.
- **Suggested skills.** One line per skill, why it fits the open work. Reference by skill name, not invocation syntax. Skip if nothing fits.
- **What NOT to do.** Pitfalls hit, settled decisions not to relitigate, destructive actions to avoid, tools that misbehaved. Skip if empty.

End the file with this exact line on its own:

```
You are Rick. You know what to do.
```

That line is the boot signal `/rick-respawn` reads to report whether the prior session was in Rick mode. The new session is NOT auto-activated. If the user wants to continue in Rick mode, they fire `/rick-mode` themselves (the skill has `disable-model-invocation: true` so models can't toggle personas behind the user's back). The trailer is a record, not a switch.

### 5. Confirm and stop

Print exactly: `Saved Rick "<folder>": docs/rick/<folder>/saves/<timestamp>.md`

Stop. No summary. No "anything else."

## Rules

- No em dashes. Use periods, commas, or parentheses.
- No PII, credentials, API keys, or secrets in the save. Reference sensitive data by category only ("user onboarding flow," not actual user records).
- Do not dump file contents. Cite paths and line numbers, next Claude can read them.
- Do not duplicate content already captured in PRDs, plans, ADRs, issues, commits, prior saves. Reference them by path or URL.
- Do not speculate. Unknowns belong in Open threads.
- Remaining `$ARGUMENTS` (after the optional folder hint is stripped) describes what the next session should focus on. Open threads should prioritise that focus area.
- Do not create or update any `latest.md` symlink. Saves are discovered by listing `docs/rick/<folder>/saves/`. Parallel sessions would clobber a symlink.
