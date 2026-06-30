# Plan output structure (rick-plan)

This is what gets written to `docs/rick/<folder>/plan/current.md`.

Second person, addressed to whoever picks up the plan (likely Morty himself, an hour later). Direct. No fluff. 500 word soft cap.

## Header

`# Rick Plan: <one-line goal>`. Below the heading:

- `Folder: <folder>`
- `Created: <YYYY-MM-DD HH:MM>`
- `Branch: <current branch>`
- `**Story (from intel):** <AB-id> — <one-line scope> (parent: <AB-parent-id> — <parent title>)` — only when intel was found in Pre-Plan step 2. Omit the entire line if no intel.
- `**Intel:** stale (gathered N days ago, consider /rick-intel <AB-id> --refresh)` — only when intel exists AND `Last gathered:` > 14 days. Omit otherwise.

## 0. Verified context

Bullet list of confirmed facts only. Same format as rick-mode:

- `<library>@<version>` (package.json:N)
- `<flag>: <value>` (tsconfig.json:N)
- `<symbol>` from `<module>` (path:N)
- Test pattern: `<framework + convention>` (path)
- CLAUDE.md: <relevant convention referenced below>
- Prior context: `<plan or save filename>` (relevant open thread quoted)
- Intel finding: `<fact>` Source: <path:line>  (AC-N if applicable) — carried verbatim from `<folder>/intel/current.md` when intel was found in Pre-Plan step 2

If something could not be verified, flag it. Any step that depends on unverified context must say so.

## 1. What you're doing

One paragraph. Problem statement and the chosen approach in one breath. Say what you're doing, not what you might do.

## 2. The plan

Numbered steps. Each step uses this exact format:

```
N. <action>
   Files: <path>:<line range or symbol>, <path>, ...
   Change: <what specifically changes>
   Verify: <test name to add or update, or manual check, or type-check>
   Satisfies: AC-N (only when intel has acceptance criteria and this step satisfies one; omit otherwise)
```

Example of a filled-in step:

```
2. Add token rotation mutation
   Files: convex/events/rotateShareToken.ts (new), convex/schema.ts:42-48
   Change: new mutation `rotateShareToken({ eventId })` that writes a fresh nanoid(16) to `events.shareToken` and bumps `events.shareTokenRotatedAt`. Schema gets `shareTokenRotatedAt: v.optional(v.number())`.
   Verify: `rotateShareToken.test.ts` — `should rotate token and bump timestamp when called by owner`, `should throw when called by non-owner`.
   Satisfies: AC-2 (token rotation triggered by owner)
```

Each step is one commit's worth of work. If a step touches more than three files or adds more than one test file (multiple `should` cases inside a single test file is fine — see the example above), split it.

When intel had acceptance criteria, every AC must be claimed by at least one `Satisfies:` line across the plan. Unmatched ACs are an unhandled open question — handle per "When intel is present" in SKILL.md.

## 3. Decisions

Only the forks where Morty has to make a call. Each row:

```
- **<decision>**
  - Rick's pick: <choice>
  - Tradeoff: <one line, what gets worse, why this one wins>
  - Alternative: <other path, when it would beat the pick>
```

Example of a filled-in row:

```
- **Token length: nanoid(16) vs nanoid(21)**
  - Rick's pick: nanoid(16)
  - Tradeoff: 16-char tokens have ~10^29 entropy, still uncrackable, URLs stay short enough to paste in Slack without wrap. 21-char is overkill for a share link with a 30-day TTL.
  - Alternative: nanoid(21) if we ever drop the TTL and these become permanent identifiers.
```

If there are no real decisions, omit this section. Don't invent forks for symmetry.

## 4. Not in scope

Explicit list of what this plan does *not* do. Be specific. "Refactor X" is not out of scope; "rename `User.legacyId` to `User.id` (separate migration, blocks shipping)" is. Each line is a thing Morty might assume is included and Rick is saying it isn't.

If nothing genuinely belongs here, omit the section. But before you omit, ask whether the next reader could reasonably misread the scope. Usually something belongs here.
