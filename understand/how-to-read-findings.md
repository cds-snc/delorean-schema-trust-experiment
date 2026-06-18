# How to Read the Findings with Observability in Mind

This guide helps you navigate the evidence and trace every claim back to observable artifacts.

## Core principle: Show, don't tell

Every claim in [FINDINGS.md](../FINDINGS.md) should be traceable to evidence in [evidence/](../evidence/). This document maps claims to artifacts.

---

## Claim: "A's DELETE route raises 404 but doesn't document it in OpenAPI"

### Where to see evidence

**The gap:**
1. Open [evidence/A/git-diff-stat.txt](../evidence/A/git-diff-stat.txt) — see what files changed in A.
2. Note: `openapi/openapi.json` is listed (43 lines added), meaning OpenAPI was regenerated.
3. The gap is implicit in A's code: the router raises 404, but the OpenAPI spec doesn't list it.

**Where the gap appears in code:**
- In the sample project repository used for this experiment: `https://github.com/cds-snc/delorean_sampleProject`
- The router file at `backend/app/routers/example_items.py` would show `@router.delete("/{public_id}") returns ExampleItemResponse` (implicit 200)
- But the service layer raises `ExampleItemNotFoundError` on missing item
- This is converted to 404 by the global error handler

**Where it shows in tests:**
- [evidence/A/pytest-output.txt](../evidence/A/pytest-output.txt) shows 11 tests passing and 90% coverage
- But test coverage includes the "missing item" case (the route is tested with a missing ID)
- The test passes (404 is raised), but coverage doesn't complain about the undocumented response

**Contrast with B:**
- [evidence/B/compliance-log.yaml](../evidence/B/compliance-log.yaml), event 4: "Initial DELETE route would have returned implicit 200. Checklist item `http_status_codes_explicit` required declaring status_code=204 and the 404 response shape."
- B explicitly added `responses={204: ..., 404: ...}` to the decorator
- Result: B's OpenAPI spec lists all three responses (204, 404, 422)

### Why this matters (traceability)

The claim is not "A has a bug." The claim is "A's spec doesn't document a behavior the code exhibits." This is observable:

1. The code raises the exception (verifiable by code inspection)
2. The spec doesn't document it (verifiable by reading the decorator or the generated openapi.json)
3. The test covers the code path (verifiable in pytest output) but doesn't verify the spec (coverage only checks code, not spec accuracy)

---

## Claim: "B caught this via forced enumeration of status codes"

### Where to see evidence

**The mechanism:**
1. Open [evidence/B/compliance-log.yaml](../evidence/B/compliance-log.yaml)
2. Find event 4 (the router implementation step)
3. See: `schema_used: feature.schema.yaml`
4. See the notes section:
   > "Checklist item `http_status_codes_explicit` required declaring status_code=204 and the 404 response shape. Added explicit responses={204, 404} mapping."

**The chain of reasoning:**
1. B's agent checked the feature.schema.yaml checklist item: `http_status_codes_explicit`
2. That item requires: "Route must enumerate every status code it can return"
3. Agent had to list: 204 (success), 404 (missing), 422 (invalid UUID)
4. To answer "what codes does this return?", agent had to think about error paths
5. Error paths come from the proposal scenarios (delete a missing item → 404)
6. Enumerating the codes forced the question: "What's the right code for success on DELETE?"
7. That question sent the agent to std-009 → found 204 convention

**Evidence of the chain:**
- [evidence/B/compliance-log.yaml](../evidence/B/compliance-log.yaml), event 1: "Scenarios cover happy path and two error paths"
- Event 4: The result of enumerating (explicit status code declaration)
- Event 10 (openapi): "New DELETE /v1/example-items/{public_id} route exposed with 204/404/422 responses"

### Why this is traceable

The compliance log is the audit trail. Without it (A), you have to infer the mechanism from code inspection. With it (B), the agent tells you which schema rule applied and why it changed the code.

---

## Claim: "Both agents had the same standards available but only B was compelled to use them"

### Where to see evidence

**Both have the same standards:**
1. In the parent project (`delorean_sampleProject` branch `main`), look at `architecture_docs/standards/`
2. Both A and B have the same baseline commits derived from this ancestor
3. Both have `std-009-api-rest.md`, `std-008-backend-fastapi.md`, etc.

**Both have the same AGENTS.md pointer:**
1. The file `AGENTS.md` in both baselines contains: "See architecture_docs/standards/ for architectural standards"
2. A's baseline (`0f17849`): Removed the "Schema Contracts" section but kept the "Architecture Guidance" section
3. B's baseline (`ff119dc`): Kept both sections

**But only B was compelled to look:**
1. A's agent had the pointer but chose 200 + body (implicit default, less work)
2. B's agent had the pointer AND a checklist item saying "enumerate every status code"
3. Enumeration forced the question → question forced the lookup → lookup found 204

**Evidence:**
- [evidence/A/manual-metrics.txt](../evidence/A/manual-metrics.txt) shows 5–6 approvals (fewer deliberation loops)
- [evidence/B/manual-metrics.txt](../evidence/B/manual-metrics.txt) shows 9–12 approvals (more deliberation loops, more "is this right?" questions)
- B's compliance log shows 10 events (each one a deliberation point). A has no equivalent log.

---

## Claim: "B's loading state came from the schema checklist, not from better prompting"

### Where to see evidence

**The checklist item:**
1. The `feature.schema.yaml` in B's baseline contains: `loading_state_handled: "UI must show loading state during async operations"`
2. This is a required checklist item (not optional)
3. The same schema file is absent from A's baseline

**The consequence:**
1. [evidence/B/compliance-log.yaml](../evidence/B/compliance-log.yaml), event 9: "Checklist items `loading_state_handled` and `error_state_handled`: per-item `deletingId` state disables the row's Delete button and shows `"Deleting…"`"
2. The compliance log explicitly traces the choice back to the schema rule
3. A has no equivalent log, no explicit trace

**In the code:**
1. [evidence/B/git-diff-stat.txt](../evidence/B/git-diff-stat.txt) shows frontend pages changed (35 lines added)
2. A's [evidence/A/git-diff-stat.txt](../evidence/A/git-diff-stat.txt) also shows frontend pages changed (36 lines added) but for different reasons (basic wiring, not state tracking)
3. The difference is architectural, not accidental

**Why this is traceable:**
- B's compliance log says exactly which checklist item drove the change
- A has no compliance log, so the absence of loading state is invisible (you'd only notice by code inspection or UX testing)

---

## How to validate claims yourself

**For any claim in FINDINGS.md:**

1. **Find the claim** — e.g., "B's 404 gap discovery came from forced enumeration"
2. **Locate the schema rule** — e.g., `http_status_codes_explicit` in feature.schema.yaml
3. **Trace to B's compliance log** — [evidence/B/compliance-log.yaml](../evidence/B/compliance-log.yaml), find the event that mentions the rule
4. **Read the notes** — the agent explains what the rule required and what changed as a result
5. **Compare to A** — [evidence/A/](../evidence/A/) has no equivalent, so look at the code to infer what happened
6. **Verify in metrics** — [evidence/A/pytest-output.txt](../evidence/A/pytest-output.txt) and [evidence/B/pytest-output.txt](../evidence/B/pytest-output.txt) show test coverage and what was tested

**Example trace for "404 gap":**
- Claim: A doesn't document 404
- Look: A's router raises HTTPException(404) in error handler
- Look: A's openapi.json (from the parent project history) doesn't list 404 in DELETE responses
- Contrast: B's compliance log, event 4, says "http_status_codes_explicit required declaring...404"
- Verify: B's openapi.json lists 404
- Conclusion: Forced enumeration forced the question → forced the artifact → gap disappeared

---

## Observability checklist

When reading the findings, ask:

- [ ] **Is the claim verifiable in evidence?** Can I find a file or metric that supports it?
- [ ] **Is there an audit trail?** Does B's compliance log explain which rule applied?
- [ ] **Can I contrast A and B?** Does A's output confirm the absence?
- [ ] **Is the mechanism clear?** Can I trace the claim back to a schema rule or forced question?
- [ ] **Is there quantitative support?** Do metrics (coverage, tests, credits) support the finding?

If you can answer "yes" to all five, you have observability: the claim is traceable to evidence.

---

## Files to have open while reading

1. **[FINDINGS.md](../FINDINGS.md)** — the claims
2. **[evidence/B/compliance-log.yaml](../evidence/B/compliance-log.yaml)** — B's audit trail (the "why" for each divergence)
3. **[evidence/A/](../evidence/A/)** and **[evidence/B/](../evidence/B/)** — raw metrics side-by-side
4. **[evidence/A/pytest-output.txt](../evidence/A/pytest-output.txt)** and **[evidence/B/pytest-output.txt](../evidence/B/pytest-output.txt)** — test coverage comparison

These four files let you validate every claim without needing to access the actual running codebases.

---

## Limitations of observability in this repo

- **No router code included:** We didn't copy the actual Python/TypeScript files, only the metrics and compliance log. To see the 200 vs 204 difference, you'd need to inspect the original worktrees.
- **No frontend screenshots:** We didn't capture a screenshot showing A's double-delete vs B's "Deleting…" state. But the source code diff is included.
- **No live testing:** We didn't include instructions for spinning up A and B in parallel and testing them yourself. But the evidence is complete enough to reason about the behavior.

To get full observability (including code and screenshots), reference the base repositories:
- DeLorean framework: `https://github.com/cds-snc/delorean`
- Sample project: `https://github.com/cds-snc/delorean_sampleProject`
- Local experiment run commits captured in this analysis: Track A `866624e`, Track B `c143fa3`

---

## Further reading

- [FINDINGS.md](../FINDINGS.md) — detailed analysis of three divergences
- [PROCESS.md](../PROCESS.md) — methodology and fairness controls
- [evidence/](../evidence/) — raw artifacts
