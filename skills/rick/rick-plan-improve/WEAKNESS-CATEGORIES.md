# Weakness categories (rick-plan-improve)

Every finding must cite one of these categories.

## Specification gaps

- **Missing error path.** Plan specifies happy path but not what happens when an outbound call times out, returns garbage, or sends a 200 with an error body. Every outbound call needs: exception thrown, what gets logged, what the caller sees.
- **Ambiguous type.** Field described as "string" when it matters whether it's ISO 8601, Unix epoch, nullable, or an enum. `Record<string, unknown>` needs a comment: intentional or lazy?
- **Unverified claim.** Plan says "the API returns X" with no spec line, response sample, or test result cited. Claims without evidence become bugs when the API returns Y.
- **Missing validation.** Input crossing a trust boundary with no validation specified. What are the constraints? What happens on violation?
- **Implicit dependency.** Plan assumes a service, config value, schema field, or table exists without checking it actually does in the codebase.

## Engineering hazards

- **Wrong implementation order.** Step 3 depends on step 5. Dependency order must be explicit and acyclic.
- **Untestable spec.** Behavior described with no test case, or a test case that does not verify what it claims to. Every non-trivial behavior needs a test name: `should [behavior] when [condition]`.
- **Missing concurrency / race condition.** Two callers hit shared mutable state and the plan does not say how conflicts are handled.
- **Config without defaults or validation.** Env var mentioned with no: required/optional flag, default value, validation rule, or startup behavior if missing.
- **Scope leak.** Plan says "out of scope" but describes behavior that requires it.
- **Scope collapse.** Critique removes a deliverable (doc, code, test) without verifying the underlying work is still covered elsewhere. Changing the *format* of a deliverable is valid. Removing it without redirecting the work is not. Before killing a deliverable, answer: "does the work this represented still need to happen? If so, where?"

## Convention violations

- **CLAUDE.md violation.** Plan contradicts a rule in `CLAUDE.md` or equivalent project instructions. Check naming, file structure, branching rules, test discipline, design-system usage.
- **Pattern divergence.** Plan invents a pattern when one already in the codebase solves the same problem.
- **Missing audit trail.** Plan creates or modifies records that need tracking (createdBy, updatedAt, soft-delete columns) without addressing them.

## Document quality

- **Redundant sections.** Two sections say the same thing. Engineers read one, miss the nuance in the other, build something satisfying neither.
- **Stale reference.** A file path, class name, or method name cited in the plan does not exist in the current codebase.
- **Missing decision rationale.** Design choice with no reason given. Engineers who don't know why will revert it later.
