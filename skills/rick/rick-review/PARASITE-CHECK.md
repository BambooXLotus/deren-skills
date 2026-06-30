# Parasite check — verification budget

Called from SKILL.md Step 6 (skepsis: cite check), applied to findings that survived the pre-existing code filter.

## Per-severity budget

| Severity | Allowed verification                                                                          |
| -------- | --------------------------------------------------------------------------------------------- |
| P0       | Batched grep, plus deep file read on grep match. Only tier that gets a full read.             |
| P1       | Batched grep only. Grep miss → kill. No file read.                                            |
| P2       | Citation existence check from `/tmp/rr-files.txt`. Kill if file isn't in the diff.            |
| P3       | No grep, no read. Kill if cited file isn't in `/tmp/rr-files.txt`. Otherwise pass to Step 7.5. |

## Batched grep (P0 + P1)

One pattern per finding, derived from the cited code snippet or named symbol — **not** a loose keyword like the function name alone.

```bash
grep -nE '<pat1>|<pat2>|<pat3>' <file1> <file2> <file3>
```

## P0 deep read

For each P0 grep match, read the file and confirm:

1. The cited code does what the finding claims.
2. Existing validation / guards / catch blocks don't already handle it.
3. Severity matches blast radius. Downgrade if not.
4. Contradictions between agents — read the code, kill the parasite.

## Marks

- **REAL** — verified, stays.
- **PARASITE** — claim doesn't match code, or pre-existing, or already mitigated. Remove silently from the report.
- **DOWNGRADE** — real issue, wrong severity. P0 only (lower tiers don't read code, so severity stays).
