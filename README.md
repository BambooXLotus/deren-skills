# deren-skills

My personal collection of Claude Code skills.

## Install

Uses the [skills.sh](https://skills.sh) installer:

```bash
npx skills@latest add BambooXLotus/deren-skills
```

You'll be prompted to pick which skills to install and which agent (Claude Code, Codex, etc.) to install them on.

To update later, re-run the same command — the installer will refresh existing skills.

## What's in here

### Rick skills (`skills/rick/`)

**The idea.** In *Rick and Morty*, Rick exists across a multiverse as countless parallel versions, each specialised — Pickle Rick is a different Rick from Doofus Rick is a different Rick from Rick C-137. When a Rick dies, a backup respawns from a save. When one Rick is stuck, he hands off to another. The rick-* skills are that idea applied to coding with an agent: each skill is a different Rick, good at one specific thing, able to hand work off to the next Rick and resume from a save. The agent is Rick; you're Morty — the one who needs Rick to figure it out, and the one Rick is brutally honest with because pretending otherwise gets Morty killed.

**How the metaphor maps.** A feature's full thread lives in one folder under `docs/rick/<folder>/` (typically the branch name, e.g. `48-rotate-share-token/`). Each parallel session is its own Rick from that folder's timeline. `rick-review` is the most literal multiverse mapping: seven named Ricks and Mortys from the show review your diff in parallel — Rick C-137, Doofus Rick, Smart Morty, Rick Prime, Pickle Rick, Scientist Rick, Evil Morty — each scoped to what they're good at (architecture, data layer, security, tests). `rick-recap` deliberately *isn't* a Rick: it's the Audit Observer from S7E6 "Rickfending Your Mort," because grading the day needs a voice that isn't already in the trenches.

**The loop.** Pick an issue. `/rick-plan <issue-number>` reads the code and writes an opinionated plan to `<folder>/plan/current.md`. You implement against the plan. `/rick-save` writes a dense briefing to `<folder>/saves/<timestamp>.md` and the session ends — that Rick "dies." Tomorrow, `/rick-respawn` boots a fresh Rick from that save with full context of where the last one left off. When the PR's ready, `/rick-review` runs the council; `/rick-fix` resolves their findings with TDD or surgical edits. At day's end, `/rick-recap` audits what actually shipped vs what was just motion. Same `<folder>` threads the whole loop — a feature's plan, saves, reviews, and recaps stay co-located.

**Why this exists.** Working with Claude on a real codebase hits two failure modes: sessions are ephemeral (conversation context dissolves when you close the tab; the next Claude has no idea what was decided), and Claude defaults to agreeable hedging, which is poison for plan critique and PR review. The rick-* skills add (1) an on-disk thread that survives session death so multi-day work doesn't restart from zero, and (2) a brutal-honesty persona that strips the hedging when you need a real opinion. `rick-improve` is meta — it edits the skill prompts themselves, versioned under `skills/rick/<name>/versions/`, so the Ricks can improve their own Rick.

All TypeScript-flavored — no NestJS or TypeORM lock-in.

| Skill | Description |
| --- | --- |
| `rick-plan` | Plan a piece of work in Rick's voice. Reads the code first, writes an opinionated numbered plan to `docs/rick/<folder>/plan/current.md`, surfaces decisions (with Rick's pick + tradeoff), and an explicit out-of-scope list. |
| `rick-plan-improve` | Bounded 3-round improvement loop on the canonical plan. Snapshots pre/post alongside the canonical at `docs/rick/<folder>/plan/v<N>-pre.md` and `v<N>.md`. Critiques against weakness categories (specification gaps, engineering hazards, convention violations, document quality), surgical edits only, 500-line cap. Explicit invocation only. Pairs with `rick-plan` like `rick-fix` pairs with `rick-review`. |
| `rick-save` | Captures the current warm thread as `docs/rick/<folder>/saves/<ts>.md`. Folder = current branch name. Writes a dense second-person briefing for the next session. |
| `rick-respawn` | Boots a new session from the most recent (or named) save. Summarizes situation, last move, blockers, and Rick mode. |
| `rick-mode` | Activate the Rick Sanchez persona for brutal architectural review. Pre-Review Protocol (read the file, trace deps + type surface, verify `package.json` + `tsconfig.json`), TS escape-hatch hunting (`as any`, `!`, `@ts-ignore`), Expected / Actual / Consequence findings. Explicit invocation only. |
| `rick-improve` | Self-improvement loop on any rick-* skill prompt. Bounded 3-round critique-and-fix, 250-line cap. Snapshots to `<skill>/versions/v<N>-pre.md` and `v<N>.md`, appends to `VERSIONS.md`. Uses `rick-mode` as the reference bar. Explicit invocation only. |
| `rick-review` | Multi-agent council code review. Up to 7 specialized Rick agents review in parallel (3 always-run, 4 conditional based on diff scope), aggregated into a versioned report at `docs/rick/<folder>/review/current.md` plus `v<N>.md` and per-agent `agents/v<N>/`. Includes parasite check (verifies cited code against the diff) and skeptic pass (kills findings that don't actually matter today). |
| `rick-fix` | Reads a `rick-review` report and resolves findings sequentially. TDD route for behavioral findings (delegates to `/tdd`), direct surgical-fix route for structural ones. Runs affected tests after each fix. Updates the report with `FIXED` / `SKIPPED` / `BLOCKED` markers. Auto-resolves the report path from the current branch. Explicit invocation only. |
| `rick-recap` | End-of-day audit across today's `rick-save` files. Groups by folder, diffs the structured sections, flags real progress vs fake progress / rabbit holes / yak shaving / avoidance. Voice is the Audit Observer (S7E6 "Rickfending Your Mort"), not Rick. Cross-folder audits write to `docs/rick/_recaps/<ts>.md`; folder-filtered to `docs/rick/<folder>/recaps/<ts>.md`. |

Feature-first layout: each feature gets one top-level folder under `docs/rick/`, defaulting to the current branch name (e.g. `48-rotate-share-token/`). Inside the folder, each artifact type owns a subdirectory: `plan/`, `review/`, `saves/`, `recaps/`. Filenames inside drop the folder prefix entirely since the directory context carries identity — `plan/current.md`, `review/v3.md`, `saves/2026-06-05_1430.md`. Cross-folder daily recaps live at `docs/rick/_recaps/` (leading underscore sorts them apart from feature folders). `rick-improve` is meta — it edits the skill prompts themselves, with versioning under `skills/rick/<name>/versions/`.

## Repo layout

```
.
├── .claude-plugin/
│   └── plugin.json             # declares the plugin and its skills
└── skills/
    └── <category>/
        └── <skill>/SKILL.md
```

## Adding a new skill

1. Create `skills/<category>/<slug>/SKILL.md`.
2. Frontmatter must include `name` (matching the folder slug) and a `description` that tells Claude *when* to trigger the skill, not just what it does. Add `disable-model-invocation: true` if it should only fire on explicit `/<slug>`.
3. Add the new path to the `skills` array in `.claude-plugin/plugin.json`.
4. Commit, push. Users get it on the next `npx skills@latest add`.

## License

MIT — see [LICENSE](./LICENSE).
