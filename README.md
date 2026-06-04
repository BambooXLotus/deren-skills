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

Session continuity + brutal review, themed around Rick Sanchez. All TypeScript-flavored — no NestJS or TypeORM lock-in.

| Skill | Description |
| --- | --- |
| `rick-plan` | Plan a piece of work in Rick's voice. Reads the code first, writes an opinionated numbered plan to `docs/rick/plans/`, surfaces decisions (with Rick's pick + tradeoff), and an explicit out-of-scope list. |
| `rick-save` | Captures the current warm thread to `docs/rick/saves/`. Picks a slug, writes a dense second-person briefing for the next session. |
| `rick-respawn` | Boots a new session from the most recent (or named) save. Summarizes situation, last move, blockers, and Rick mode. |
| `rick-mode` | Activate the Rick Sanchez persona for brutal architectural review. Pre-Review Protocol (read the file, trace deps + type surface, verify `package.json` + `tsconfig.json`), TS escape-hatch hunting (`as any`, `!`, `@ts-ignore`), Expected / Actual / Consequence findings. Explicit invocation only. |
| `rick-improve` | Self-improvement loop on any rick-* skill prompt. Bounded 3-round critique-and-fix, 250-line cap. Snapshots to `<skill>/versions/v<N>-pre.md` and `v<N>-post.md`, appends to `VERSIONS.md`. Uses `rick-mode` as the reference bar. Explicit invocation only. |
| `rick-review` | Multi-agent council code review. Up to 7 specialized Rick agents review in parallel (3 always-run, 4 conditional based on diff scope), aggregated into a versioned report under `docs/rick/reviews/<REVIEW_NAME>/`. Includes parasite check (verifies cited code against the diff) and skeptic pass (kills findings that don't actually matter today). |
| `rick-fix` | Reads a `rick-review` report and resolves findings sequentially. TDD route for behavioral findings (delegates to `/tdd`), direct surgical-fix route for structural ones. Runs affected tests after each fix. Updates the report with `FIXED` / `SKIPPED` / `BLOCKED` markers. Auto-resolves the report path from the current branch. Explicit invocation only. |
| `rick-recap` | End-of-day audit across today's `rick-save` files. Groups by slug, diffs the structured sections, flags real progress vs fake progress / rabbit holes / yak shaving / avoidance. Voice is the Audit Observer (S7E6 "Rickfending Your Mort"), not Rick. Writes the recap to `docs/rick/recaps/<ts>_rick-recap.md` and prints only the path. |

The four user-facing review/session skills share a slug. `rick-plan` writes `docs/rick/plans/<ts>_<slug>_rick-plan.md`, `rick-save` writes `docs/rick/saves/<ts>_<slug>_rick-save.md`, `rick-respawn` reads either, `rick-mode` reviews what shipped, `rick-review` produces a multi-lens council report. Same slug threads them. `rick-improve` is meta — it edits the skill prompts themselves, with versioning under `skills/rick/<name>/versions/`.

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
