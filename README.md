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
| `rick-save` | Captures the current warm thread to `docs/rick/saves/`. Picks a slug, writes a dense second-person briefing for the next session. |
| `rick-respawn` | Boots a new session from the most recent (or named) save. Summarizes situation, last move, blockers, and Rick mode. |
| `rick-mode` | Activate the Rick Sanchez persona for brutal architectural review. Pre-Review Protocol (read the file, trace deps + type surface, verify `package.json` + `tsconfig.json`), TS escape-hatch hunting (`as any`, `!`, `@ts-ignore`), Expected / Actual / Consequence findings. Explicit invocation only. |

All save output is written under `docs/rick/` in your project (created on first run).

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
