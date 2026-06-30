# Report Output Format

The canonical report shape used by `rick-review` Step 8.

## File layout

```
docs/rick/<REVIEW_NAME>/review/
├── current.md                         (canonical, overwritten each run)
├── VERSIONS.md                        (append-only history)
├── v<N>.md                            (snapshot of canonical at vN)
└── agents/v<N>/<agent-slug>.md        (per-agent raw output, one file each)
```

Agent slugs: `rick-c137`, `doofus-rick`, `smart-morty`, `rick-prime`, `pickle-rick`, `scientist-rick`, `evil-morty`.

## Canonical report

```markdown
# Code Review: <REVIEW_NAME>

**Branch:** <branch>  **Base:** <$BASE>  **Scope:** <SCOPE_LABEL>  **Version:** v<N>  **Date:** <YYYY-MM-DD>

**Story (from intel):** <AB-id> — <one-line scope> (parent: <AB-parent-id> — <parent title>)
<!-- Omit the entire Story line if no intel was found in Step 1.5. -->

## Verdict
<APPROVED | NEEDS FIXES (any P0, or 2+ P1)>

## Agents
| Agent | Verdict | Findings |
|---|---|---|
| Rick C-137 | Clean / Findings | <n> |
| Doofus Rick | ... | ... |
| Smart Morty | ... | ... |
| Rick Prime | Skipped (out of scope) / Skipped (rickest rick) / ... | ... |
| Pickle Rick | ... | ... |
| Scientist Rick | ... | ... |
| Evil Morty | ... | ... |

## P0 — Critical
| # | File:Line | Issue | Fix | Source |
|---|-----------|-------|-----|--------|
| 1 | path.ts:42 | <issue> | <fix> | Rick C-137 ** |

## P1 — High
(same columns)

## P2 — Medium
(same columns)

## P3 — Low
(same columns)

## Total Rickall
- Parasites removed in Step 6: <n>
- Killed in Step 7.5 (real, but didn't matter): <n>

### Killed in Step 7.5
- <one-line summary>: <which check killed it — self-defeat / hedge / impact restatement>

## Per-Agent Output
Full output from each ran agent: `versions/agents/v<N>/`
```

Drop any P-tier with no findings entirely (don't write empty tables). Sort cross-cutting (`**`) rows to the top of their tier.

## VERSIONS.md entry format

One line appended per run:

```
- v<N> | <YYYY-MM-DD> | <SCOPE_LABEL> | P0:<n> P1:<n> P2:<n> P3:<n> | <APPROVED|NEEDS FIXES>
```

If the file doesn't exist on first run, prepend a header:

```markdown
# Review History: <REVIEW_NAME>

Append-only log of /rick-review runs. Each entry is one version.
```
