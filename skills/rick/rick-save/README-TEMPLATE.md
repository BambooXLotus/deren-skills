# Rick

Artifacts from the Rick skills. Feature-first layout: each feature gets one top-level folder (branch name when available, e.g. `48-rotate-share-token/`, or a one-word slug fallback). Inside that folder, each artifact type owns a subdirectory.

## Structure

```
docs/rick/
  <folder>/                            # one per feature (branch or slug)
    plan/
      current.md                       # canonical plan (read this)
      v1.md, v2-pre.md, v2.md          # version history alongside
      VERSIONS.md                      # changelog
    review/
      current.md                       # canonical review
      v1.md, v2.md
      VERSIONS.md
      agents/v<N>/<agent>.md           # per-agent raw outputs per review version
    saves/<YYYY-MM-DD_HHMM>.md         # append-only session saves
    recaps/<YYYY-MM-DD_HHMM>.md        # folder-filtered audits (rare)
  _recaps/<YYYY-MM-DD_HHMM>.md         # cross-folder daily audits
```

## Commands

- `/rick-plan` writes `<folder>/plan/current.md` plus initial `v1.md` and `VERSIONS.md`.
- `/rick-plan-improve` runs a bounded loop on `current.md`, snapshots `v<N>-pre.md` / `v<N>.md`.
- `/rick-save` appends a timestamped save to `<folder>/saves/`. Folder defaults to current branch.
- `/rick-respawn` boots from the most recent save. Pass a folder name (`/rick-respawn 48-rotate-share-token`), a slug, or a path.
- `/rick-review` writes a multi-agent council review to `<folder>/review/current.md` plus `v<N>.md` and per-agent `agents/v<N>/`.
- `/rick-fix` reads `<folder>/review/current.md` and resolves findings sequentially.
- `/rick-recap` audits saves. No argument = cross-folder audit at `_recaps/`. Argument = folder-filtered at `<folder>/recaps/`.

All artifacts are second person, addressed to the next reader. Old files stay as an audit trail.
