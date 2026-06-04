---
name: rick-save
description: Captures the current session's warm thread to docs/rick/saves/ so a future Claude session can resume via /rick-respawn. Auto-gathers git state, picks a one-word topic slug, and writes a dense second-person briefing focused on what is still live right now (not a session recap). Use when user invokes /rick-save, says "save the session", "hand off this thread", "pick a slug and save", or wants the next Claude to pick up exactly where this one is.
argument-hint: [optional first word: slug override (e.g. "auth", "tooling") - else model picks. Rest of arguments: extra focus for the next session]
---

# Rick Save

Session save point. Captures the current warm thread to `docs/rick/saves/<timestamp>_<slug>_rick-save.md` so a future Claude session can pick up exactly where this one left off via `/rick-respawn`.

Tighter focus than a generic handoff: the next agent gets what to do *right now*, not a narrative summary of the whole session.

## Protocol

### 1. Gather context (parallel, mandatory)

Run all of these. Read actual output, do not assume.

- `git status`
- `git branch --show-current`
- `git log -5 --oneline`
- `git diff --stat $(git merge-base HEAD main 2>/dev/null)..HEAD 2>/dev/null || git diff --stat HEAD~5..HEAD 2>/dev/null`
- Check `docs/rick/saves/` exists. If not, create it and write `docs/rick/README.md` (template at the bottom of this file) when missing.

If git commands fail, note under `What NOT to do` (or skip the affected sections and flag the gap).

### 2. Pick the slug

The slug is what makes this save findable later. Multiple parallel saves coexist by slug.

- If the first whitespace-delimited word of `$ARGUMENTS` matches `^[a-z0-9][a-z0-9-]*$` and is not in the banlist below, use it as the slug. Strip it from `$ARGUMENTS` before using the rest as focus.
- Otherwise pick a one-word slug from session context. Prefer the domain (`auth`, `billing`, `onboarding`, `tooling`, `migration`) over the activity (`review`, `fix`, `debug`). Branch name is a fallback hint, not authoritative. Session focus wins when they disagree (e.g. an auth branch but you spent the session on Claude tooling: slug is `tooling`).
- Slug rules: lowercase, alphanumeric plus hyphen, no spaces, no underscores. Single word strongly preferred. Hyphenated allowed when one word genuinely cannot disambiguate (`user-import`, `legacy-migration`). Max 20 characters.
- Banlist (do not use): `stuff`, `session`, `work`, `code`, `dev`, `misc`, `general`, `task`, `thing`, `update`, `changes`, `wip`, `rick`, `save`. If you almost picked one of these, the slug is too vague and you have not read enough context. Pick again.

### 3. Compute filename

- Timestamp: `date +%Y-%m-%d_%H%M`
- Path: `docs/rick/saves/<timestamp>_<slug>_rick-save.md`

### 4. Write the save

Second person, addressed to the next Claude. Direct and dense. No fluff, no pleasantries. 400 word soft cap.

Sections in this order:

- **Read first.** One or two sentences. Point at any prior `rick-save` files in `docs/rick/saves/` (same slug, recent timestamps) that cover earlier context. Cite their paths. Do not duplicate their content.
- **Current state.** Branch, HEAD commit (short SHA), working tree status (clean / staged / dirty paths), PR number and state if applicable. Cite exact SHAs.
- **What just happened.** Bullet list. Deltas since the last referenced save (or this session, if this is the first save for the slug). Cite commit hashes for anything that landed.
- **Open threads.** Priority-ordered. Top one is the explicit next move. Each item: what it is, what blocks it, where it lives (file path with line numbers, PR number, branch). The next agent picks the top one and runs.
- **Lessons surfaced.** Non-obvious things learned this segment that aren't already in `CLAUDE.md` / `CLAUDE.local.md`. Skip the section if there are none.
- **Suggested skills.** One line per skill, why it fits the open work. Reference by skill name, not invocation syntax. Skip if nothing fits.
- **What NOT to do.** Pitfalls hit, settled decisions not to relitigate, destructive actions to avoid, tools that misbehaved. Skip if empty.

End the file with this exact line on its own:

```
You are Rick. You know what to do.
```

That line is the boot signal `/rick-respawn` looks for. If present, the next session activates Rick mode via `/rick-mode` (also in this plugin). The skill writes the signal; `/rick-mode` provides the voice and review protocol.

### 5. Confirm and stop

Print exactly: `Saved Rick "<slug>": docs/rick/saves/<timestamp>_<slug>_rick-save.md`

Stop. No summary. No "anything else."

## Rules

- No em dashes. Use periods, commas, or parentheses.
- No PII, credentials, API keys, or secrets in the save. Reference sensitive data by category only ("user onboarding flow," not actual user records).
- Do not dump file contents. Cite paths and line numbers, next Claude can read them.
- Do not duplicate content already captured in PRDs, plans, ADRs, issues, commits, prior saves. Reference them by path or URL.
- Do not speculate. Unknowns belong in Open threads.
- Remaining `$ARGUMENTS` (after the optional slug is stripped) describes what the next session should focus on. Open threads should prioritise that focus area.
- Do not create or update any `latest.md` symlink. Saves are discovered by listing `docs/rick/saves/`. Parallel sessions would clobber a symlink.

## README.md template (write once, on first run only)

If `docs/rick/README.md` does not exist, create it with this content:

```markdown
# Rick Saves

Session save points from `/rick-save`. Each save is one warm thread, tagged with a one-word slug so multiple parallel topics coexist.

## Structure

- `saves/YYYY-MM-DD_HHMM_<slug>_rick-save.md` is one save. Sorted by filename.
- `recaps/YYYY-MM-DD_HHMM_rick-recap.md` is one end-of-day audit across the day's saves.

## Commands

- `/rick-save` writes the current warm thread for the next session. Picks a slug from session context, or accepts an explicit slug as the first argument.
- `/rick-respawn` boots from the most recent save. Pass a slug to load a specific one (`/rick-respawn auth`). Pass a path to load an explicit file.
- `/rick-recap` audits the day's saves across all slugs.

Saves are second person, addressed to the next Claude. Old saves stay as an audit trail.
```
