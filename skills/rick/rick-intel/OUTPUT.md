# Intel file output structure (rick-intel)

This is what gets written to `docs/rick/<folder>/intel/current.md`, where `<folder>` is the branch name (when on a feature branch) or the predicted branch name `<digits>-<title-slug>` (when run pre-branch).

## Section order

Omit `Open questions` / `Sources crawled` if empty. Never omit `Story Context` or `Scope`.

```markdown
# <AB-id> — <Story Title>

Last gathered: <YYYY-MM-DD>

## Story Context

- Parent: <AB-parent-id> (<parent title>)
- State: <state>
- Iteration: <iteration>
- Assigned: <name>
- PR: <#NNN if present>
- Depends: <if mentioned in description>
- Blocks: <if mentioned in description>

## Scope

> <verbatim ADO description, HTML stripped to prose>

Build targets:

- <bullet for each concrete noun phrase from the scope: a controller, a DTO, a CASL handler, a migration, etc.>

## AC-N — <AC title>

- <one-line finding summary> — Source: <path:line | url> — Found: <YYYY-MM-DD>
- ...

(or, if no AC parseable:)

## Findings

- <finding> — Source: <…> — Found: <YYYY-MM-DD>

## Open questions

- <auto-populated>

## Sources crawled

- <every path / URL the gather phase touched, even zero-hit ones>
```

## Finding format

```
- <one-line summary> — Source: <path:line | full URL> — Found: <YYYY-MM-DD>
```

## Refresh: stale lines

When a refresh changes a finding, strike the old and append the new:

```
- ~~<old summary>~~ — Source: <…> — Found: <orig date> — Stale: <today>
- <new summary> — Source: <same source> — Found: <today>
```

Unchanged findings keep their original `Found:` date.
