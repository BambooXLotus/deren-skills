# Documented Intent Cross-Reference (Step 7.7)

After Step 7.5 kills findings that didn't matter, this step cross-references survivors against **documented intent** — the PR description body (Step 3.5(b)) and the rick-intel dossier (Step 1.5). Both are sources of "what the author / story expected"; findings either align, contradict, or fall outside that expectation.

This is the orchestrator's final filter. No new tool calls, no agent re-launch. Operates on the post-7.5 finding tables and on `$PR_INTENT` / intel summary already in context.

## When this step fires

- `$PR_INTENT` is non-empty (Step 3.5(b) extracted a PR body), OR
- Step 1.5 found an intel dossier.

If both are empty, skip Step 7.7 entirely. No documented intent to cross-reference against.

---

## PR body source — apply against each surviving finding

### 1. Deferred-work demotion

If the PR body explicitly defers the concern the finding raises (e.g. "validation deferred to follow-up", "seed importer in a later PR", "freezeForSubmission deferred to orchestrator WP"):

- DEMOTE one tier.
- Append to the Issue cell: `**Note:** PR description defers this — "<short quote from PR body>"`.

### 2. Contradiction strengthening

If the finding *contradicts* the PR body (code does X, PR says Y will happen):

- STRENGTHEN one tier (P3 → P2, P2 → P1, P1 stays P1 with `**` cross-cutting mark).
- Append: `**Contradicts PR <section name>:** "<short quote>"`.
- These belong at the top of their tier — implementation-drifted-from-documented-intent is the highest-signal class of finding the PR body produces.

### 3. Acknowledged-with-mitigation note

If the PR body acknowledges the issue and asserts a planned mitigation not yet present in the diff:

- KEEP at current tier.
- Append: `**Mitigation noted in PR:** "<short quote>" — until that lands, the gap is active.`

### 4. Scope flag removal

If a finding flags "missing X" and the PR body explicitly carves X out ("except X", "foundation only", "X in a follow-up"):

- REMOVE the finding entirely.
- Note the removal in Total Rickall under a new sub-heading:

```
### Removed via Step 7.7 (PR explicitly defers)
- [scope flag]: PR header says "<quote>"
```

---

## Intel source — apply when Step 1.5 found a dossier

The dossier carries the story's AC, scope, and cited domain ADRs — what the *upstream story* asked for. Higher-signal than PR body because AC is a committed deliverable, not just author intent for one slice. Where PR body and intel conflict on the same finding, intel wins (the AC is the spec; the PR body is the implementation note).

### 1. AC contradiction

If a finding contradicts what an AC line in intel requires:

- STRENGTHEN one tier (P3 → P2, P2 → P1; P1 stays P1 with `**`).
- Append: `**Contradicts AC-<N>:** "<one-line AC text from intel>"`.
- Strongest finding class — implementation drifted from the spec.

### 2. AC scope exclusion

If a finding flags "missing X" and intel's Story Context or AC notes carve X out of the story's scope (e.g. "ships per-name extensions; #9 lands the third metric"):

- DEMOTE one tier (P3 → KILL).
- Append: `**Out of intel scope:** "<short quote from intel>"`.

### 3. ADR contradiction

If a finding contradicts a settled ADR that intel cited as a domain anchor:

- DEMOTE one tier (P3 → KILL).
- Append: `**Settled by ADR-<N>:** "<short quote from intel finding>"`.

---

## Conflict resolution

Multiple rules can apply to the same finding (e.g. PR body defers AND intel scope excludes). Apply all that fit, take the most aggressive outcome:

```
KILL trumps DEMOTE trumps STRENGTHEN
```

If a finding gets both STRENGTHEN (from AC contradiction) and DEMOTE (from PR body deferral), the STRENGTHEN wins — AC is the spec, PR body's deferral doesn't override the committed deliverable.

## Output

Re-number tiers after all demotions / strengthenings / removals. Step 8's tier counts (printed to chat, written to VERSIONS.md) are the post-7.7 numbers.

Audit trail in Total Rickall:

```
### Adjusted via Step 7.7 (documented-intent cross-reference)
- [finding]: STRENGTHENED to P<N> — contradicts AC-<N>
- [finding]: DEMOTED to P<N> — PR body defers
- [finding]: REMOVED — PR explicitly scopes out
```
