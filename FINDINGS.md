# Findings: Three Real Divergences

Date: 2026-06-18

Each divergence is analyzed below with:
- What changed between A and B
- Why it matters
- The mechanism that caused it (traced back to schema rules)
- Observability (where to see evidence in this repo)

---

## Divergence 1: OpenAPI contract completeness

### What changed

| | A (unguided) | B (schema-guided) |
|---|---|---|
| **DELETE /v1/example-items/{public_id} documented responses** | `{200, 422}` | `{204, 404, 422}` |
| **Code behavior** | Raises `HTTPException(404)` for missing items | Declares `responses={204: ..., 404: ...}` |
| **Gap** | 404 is raised in code but not documented in spec | No gap |

### Why this matters

A's OpenAPI spec does not document the 404 response, even though the code raises it. This creates cascading problems:

1. **SDK generation:** Any typed client generated from A's spec will not know to handle a 404 response as a distinct error state.
2. **Contract mismatch:** Consumers read the spec and expect only `{200, 422}`; a 404 response surprises them.
3. **Invisible gap:** Unit tests pass (the route still raises 404). Linting, type-checking, and test coverage don't catch the undocumented exception. The gap only surfaces when an external consumer implements against the spec weeks or months later.

B's contract is complete: every response status code the code can produce is documented.

### The mechanism: Forced enumeration via schema checklist

B caught this because of a three-link chain in the schema corpus:

**Link 1: Requirements schema (`prd.schema.yaml`)**
```
scenarios:
  - happy path (create, list, delete)
  - error path: delete a missing item → 404
```

B's proposal listed both. A's did too (they used the same proposal text).

**Link 2: Feature schema (`feature.schema.yaml`) — checklist item `http_status_codes_explicit`**
```
required_checklist_items:
  - http_status_codes_explicit: "Route must enumerate every status code it can return"
```

This checklist item forced B's agent to answer: "What status codes does the DELETE route return?" To answer that, the agent had to:
1. Trace the service logic (raises `ExampleItemNotFoundError` on miss)
2. Trace the error handler (converts to 404)
3. Trace the router decorator (declares `status_code=204` + `responses={404: ...}`)

Result: B's compliance log explicitly credits this rule:
> "Checklist item `http_status_codes_explicit` required declaring status_code=204 and the 404 response shape. Added explicit `responses={204, 404}` mapping."

**Link 3: Feature schema — checklist item `openapi_check_passes`**
```
required_checklist_items:
  - openapi_check_passes: "make check-openapi must report 'OpenAPI file is current'"
```

This forced B to run `make export-openapi` and verify the 404 is present in `openapi/openapi.json`. The 404 flows from the router declaration → the OpenAPI generation.

A was never compelled to enumerate status codes, so A never opened the checklist item, never traced the error paths, and never noticed the gap.

### Observability: Where to see evidence

- **B's compliance log:** See [evidence/B/compliance-log.yaml](evidence/B/compliance-log.yaml), implementation event for `backend/app/routers/example_items.py`: "Initial DELETE route would have returned implicit 200. Checklist item `http_status_codes_explicit` required declaring status_code=204 and the 404 response shape."
- **B's test coverage:** [evidence/B/pytest-output.txt](evidence/B/pytest-output.txt) shows 12 tests passed; the new 404 test is included.
- **A's gap:** [evidence/A/pytest-output.txt](evidence/A/pytest-output.txt) shows 11 tests passed; test for 404 exists but spec gap is invisible to coverage.
- **Metric:** A changed 9 files, B changed 11 files (the extra 2 are compliance log updates and test file additions).

---

## Divergence 2: HTTP DELETE semantics

### What changed

| | A (unguided) | B (schema-guided) |
|---|---|---|
| **Service `delete_item` returns** | `ExampleItemRead` (the deleted item) | `None` |
| **Router declares** | `response_model=ExampleItemResponse` | `status_code=204, response_class=Response` |
| **Wire response** | `200 OK + JSON body` of the deleted item | `204 No Content + empty body` |

### Why this matters

Both are internally consistent (A's frontend successfully parses the JSON it receives). But externally:

1. **HTTP intermediaries:** Caches, proxies, and CDNs treat `200 + body` and `204` differently. A's pattern surprises standard infrastructure.
2. **REST idiom:** HTTP convention (STD-009, "REST API") prescribes `204 No Content` for successful destructive operations. A's `200 + body` is non-idiomatic.
3. **External consumers:** Any SDK generated from A's spec, or any external team implementing against the API, will read `200` and expect a response body. A violates that expectation.

### The mechanism: Forced enumeration via same checklist

**Link 1: Feature schema — same checklist item `http_status_codes_explicit`**

B's agent was compelled to enumerate status codes. That required answering: "What status code should DELETE return?"

The feature spec (from both A and B) says "soft-delete an item." The agent needed to pick a status code. A reached for what worked: return the deleted object, get 200 automatically. B was forced to declare an explicit code, which triggered a deliberate lookup.

**Link 2: Architecture standards (both tracks had this)**

Both A and B have access to [architecture_docs/standards/std-009-api-rest.md](https://github.com/cds-snc/delorean_sampleProject/blob/main/architecture_docs/standards/std-009-api-rest.md): 
> "Successful DELETE operations return `204 No Content` (no response body) per RFC 7231."

**Link 3: The question that forced the lookup**

A's agent had the standard on disk but wasn't compelled to consult it. B's checklist item forced the agent to declare the status code explicitly, which prompted the question: "What's the REST convention for DELETE?" That question led to std-009.

Both `AGENTS.md` files contain identical pointers to the standards directory. The difference is not "B had better prompts." The difference is **B was compelled to enumerate**, which turned a passive "have access to" into an active "must answer this question."

B's compliance log credits the mechanism explicitly:
> "Checklist item `http_status_codes_explicit` required declaring status_code=204 and the 404 response shape. Added explicit `responses={204, 404}` mapping and `response_class=Response` so the route honours the no-body contract."

### Observability: Where to see evidence

- **B's compliance log:** [evidence/B/compliance-log.yaml](evidence/B/compliance-log.yaml), event 4: The step-by-step trace of how the 204 choice was made.
- **Code diff:** Compare the router files in each track:
  - A: `response_model=ExampleItemResponse` (implicit 200)
  - B: `status_code=204, response_class=Response` (explicit)
- **Metrics:** Both A and B have 11+ tests passing; the difference is architectural choice, not test coverage.

---

## Divergence 3: UX in-flight state

### What changed

| | A (unguided) | B (schema-guided) |
|---|---|---|
| **Delete button state** | None (always clickable) | Tracks `deletingId`, disables during deletion |
| **User experience** | Double-click fires two DELETE requests | Double-click: first request disables button, second click is blocked |
| **Loading label** | None | Button label switches to "Deleting…" |

### Why this matters

A allows accidental duplicate deletes:

1. **UX defect:** User clicks Delete, sees no visual feedback, clicks again. Two DELETE requests fire in parallel.
2. **Race condition:** The second request may succeed or fail depending on timing.
3. **Correction burden:** The second request is at least wasted work and can surface avoidable API errors.

B prevents this by disabling the button and showing "Deleting…", so the second click does nothing.

### The mechanism: Forced enumeration via schema checklist

**Link 1: Feature schema — checklist item `loading_state_handled`**
```
required_checklist_items:
  - loading_state_handled: "UI must show loading state during async operations"
```

B's agent was compelled to add loading state. That required identifying every async operation in the feature and tracking state for each. For DELETE, that meant:
1. Adding per-item `deletingId` state
2. Disabling the button while deletion is in flight
3. Showing "Deleting…" label

A was never compelled to add this, so A skipped it.

**Link 2: Same checklist — `error_state_handled`**
```
required_checklist_items:
  - error_state_handled: "UI must show error feedback and allow recovery"
```

This reinforced the need for state tracking and error handling in the UI layer.

### Observability: Where to see evidence

- **B's compliance log:** [evidence/B/compliance-log.yaml](evidence/B/compliance-log.yaml), event 9: "Checklist items `loading_state_handled` and `error_state_handled`: per-item `deletingId` state disables the row's Delete button and shows `"Deleting…"`."
- **Code diff:** [evidence/B/git-diff-stat.txt](evidence/B/git-diff-stat.txt) shows the frontend page changed (35 lines added for state tracking).
- **A's gap:** [evidence/A/git-diff-stat.txt](evidence/A/git-diff-stat.txt) shows the frontend page changed (36 lines added) but for different reasons (basic UI wiring, not state tracking).
- **Metrics:** B added more tests on the frontend side (2 new tests for `apiDelete` function). A did not.

---

## Summary: One mechanism, three manifestations

All three divergences flow from the same root cause: **schemas force enumeration.**

When B's agent was compelled to:
- Enumerate every status code → discovered the 404 gap + standard convention for 204
- Enumerate loading states → added in-flight state tracking

A's agent was not compelled, so A relied on implicit defaults and happenstance.

This is not a statement about the model's capability or knowledge. Both used the same model, same codebase, same standards. The difference is **compulsion level.**

- Passive access: "You can read STD-009 if you want."
- Active compulsion: "You must answer: what status codes does your route return? List them all."

The second produces better outcomes because enumeration forces deliberation.

---

## What this does NOT prove

- **Not a general measure of AI quality.** This is one feature, one moment, one model.
- **Not about long-term maintainability.** We can't speak to drift over months or handoff dynamics.
- **Not about code aesthetics.** Both implementations are valid; A is not "bad code," it's just uncompelled code.
- **Not about subjective preferences.** We did not run a code review with humans and ask "which is prettier?"

What this *does* show: when you compel enumeration, you catch gaps that passive access misses.

---

## Next steps

- Read [PROCESS.md](PROCESS.md) to understand how we kept the comparison fair.
- Dig into [evidence/B/compliance-log.yaml](evidence/B/compliance-log.yaml) to see the detailed trace of what B produced.
- Compare [evidence/A/pytest-output.txt](evidence/A/pytest-output.txt) and [evidence/B/pytest-output.txt](evidence/B/pytest-output.txt) to see test coverage differences.
