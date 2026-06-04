# deren-skills

A Claude Code plugin marketplace containing my personal skills.

## Install

In Claude Code, add this repo as a marketplace, then install the plugin:

```
/plugin marketplace add BambooXLotus/deren-skills
/plugin install deren-skills@deren-skills
```

The first `deren-skills` is the plugin name, the second is the marketplace name.

To update later:

```
/plugin marketplace update deren-skills
```

## What's in here

| Skill | Description |
| --- | --- |
| `hello-deren` | Smoke-test placeholder. Remove once you've added real skills. |

## Repo layout

```
.
├── .claude-plugin/
│   └── marketplace.json        # catalog Claude Code reads
└── plugins/
    └── deren-skills/
        ├── .claude-plugin/
        │   └── plugin.json     # plugin manifest (name, version)
        └── skills/
            └── <skill-name>/
                └── SKILL.md    # the skill itself
```

## Adding a new skill

1. Create `plugins/deren-skills/skills/<slug>/SKILL.md`.
2. Frontmatter must include `name` (matching the folder slug) and a `description` that tells Claude *when* to trigger the skill — not just what it does.
3. Bump `version` in `plugins/deren-skills/.claude-plugin/plugin.json`.
4. Commit, push. Users get it on `/plugin marketplace update`.

## License

MIT — see [LICENSE](./LICENSE).
