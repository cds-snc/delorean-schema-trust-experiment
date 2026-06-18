# Divergence 1: OpenAPI Contract Completeness

## The gap

**Track A:** DELETE /v1/example-items/{public_id} documents responses: `{200, 422}`  
**Track B:** DELETE /v1/example-items/{public_id} documents responses: `{204, 404, 422}`

A's router raises `HTTPException(404)` for missing items, but this response is not documented in the OpenAPI spec.

## Why it matters

1. **SDK generation:** Typed clients generated from A's spec won't know how to distinguish "not found" from network errors.
2. **External contracts:** Consumers read the spec and expect only `{200, 422}`; a 404 surprises them.
3. **Invisible to standard tools:** Unit tests pass (the route still raises 404), linting doesn't complain, coverage doesn't catch it. The gap only surfaces when an external consumer tries to implement against the spec.

## The mechanism

B caught this via a three-link schema chain:

1. **prd.schema.yaml requirement:** Feature proposal must list error scenarios
   - Both A and B listed: delete a missing item → 404
   
2. **feature.schema.yaml checklist:** `http_status_codes_explicit`
   - "Route must enumerate every status code it can return"
   - B's agent was compelled to list: 204, 404, 422
   - A's agent was not compelled
   
3. **feature.schema.yaml checklist:** `openapi_check_passes`
   - "make check-openapi must report 'OpenAPI file is current'"
   - B had to verify the 404 made it into openapi/openapi.json

A was never compelled to enumerate, so A never traced the error path, never listed 404 in the decorator, and never verified the spec was complete.

## Evidence

**B's compliance log:**  
[../../evidence/B/compliance-log.yaml](../../evidence/B/compliance-log.yaml), event 4:
> "Initial DELETE route would have returned implicit 200. Checklist item `http_status_codes_explicit` required declaring status_code=204 and the 404 response shape. Added explicit `responses={204, 404}` mapping."

**Test coverage:**  
- [../../evidence/A/pytest-output.txt](../../evidence/A/pytest-output.txt): 11 tests pass, 90% coverage. The missing-item test exists but spec gap is invisible.
- [../../evidence/B/pytest-output.txt](../../evidence/B/pytest-output.txt): 12 tests pass, 92% coverage. Extra test for 404 response in decorator.

**Metric:**  
- A: 9 files changed
- B: 11 files changed (+2 for compliance log updates and test additions)

## Conclusion

Forced enumeration forced the question "what codes does this return?", which led to complete documentation. Access to the same standards and same test suite wasn't enough; enumeration was the key.
