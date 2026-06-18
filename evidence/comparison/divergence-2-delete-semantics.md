# Divergence 2: HTTP DELETE response semantics

## The difference

| | Track A | Track B |
|---|---|---|
| Service returns | `ExampleItemRead` (the deleted item) | `None` |
| Router declares | `response_model=ExampleItemResponse` | `status_code=204, response_class=Response` |
| Wire response | `200 OK + JSON body` | `204 No Content + empty body` |

Both are internally consistent (A's frontend parses the JSON it receives), but externally they differ in REST idiom.

## Why it matters

1. **HTTP intermediaries:** Caches, proxies, CDNs treat `200 + body` and `204` differently. A violates standard assumptions.
2. **REST convention:** RFC 7231 and every REST API guide prescribe `204 No Content` for successful destructive operations (DELETE, destructive PATCH). A's `200 + body` is non-idiomatic.
3. **External consumers:** Any SDK or external team reading A's API contract will expect a response body with 200; A surprises them.

## The mechanism

Both A and B have access to the same standard: `architecture_docs/standards/std-009-api-rest.md` (REST API conventions).

But only B was forced to consult it:

1. **feature.schema.yaml checklist:** `http_status_codes_explicit`
   - "Route must enumerate every status code it can return"
   - B's agent had to declare an explicit status code
   - Question: "What status code should successful DELETE return?"
   - Answer requires consulting conventions
   - Lookup: std-009 → "204 No Content"

2. **A was not compelled to enumerate**
   - A reached for what worked: return the deleted object, implicitly get 200
   - Never opened std-009 because there was no mandatory question forcing the lookup

## Evidence

**B's compliance log:**  
[../../evidence/B/compliance-log.yaml](../../evidence/B/compliance-log.yaml), event 4:
> "Checklist item `http_status_codes_explicit` required declaring status_code=204 and the 404 response shape."

**Approval metrics:**  
- A: 5–6 approvals (fewer deliberation points)
- B: 9–12 approvals (more "is this right?" questions, more standard lookups)

B asked deliberate questions because enumeration forced decisions. A used defaults because it didn't have to enumerate.

## Conclusion

The standard was on disk for both. Prompts mentioned "follow our standards." The difference was **compulsion to enumerate**, which turned "you can read the standard" into "you must answer this question, which requires reading the standard."

Same knowledge, different decision-making process → different outcomes.
