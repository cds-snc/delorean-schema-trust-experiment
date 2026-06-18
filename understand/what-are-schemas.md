# What are schemas in this context?

"Schemas" in this experiment refer to typed YAML files that define **what must be decided** and **what artifacts must be produced** when implementing a feature.

This is not the same as JSON schemas, Pydantic models, or database schemas (though those are related). This is about **decision schemas** — rules that compel an agent to enumerate and document choices.

For direct inspection, the exact schema files used in this experiment are vendored in this repo:

- [../source/delorean/schemas/prd.schema.yaml](../source/delorean/schemas/prd.schema.yaml)
- [../source/delorean/schemas/feature.schema.yaml](../source/delorean/schemas/feature.schema.yaml)
- [../source/delorean/schemas/evidence.schema.yaml](../source/delorean/schemas/evidence.schema.yaml)
- [../source/delorean/schemas/compliance-log.schema.yaml](../source/delorean/schemas/compliance-log.schema.yaml)

Provenance: copied from https://github.com/cds-snc/delorean_sampleProject at commit `c143fa32732561ebca4d00c629687661b5eb83fc`.

## The three schemas in this experiment

### 1. prd.schema.yaml — Requirements schema

Source file: [../source/delorean/schemas/prd.schema.yaml](../source/delorean/schemas/prd.schema.yaml)

Defines what a feature proposal must contain:

```yaml
required_fields:
  - description: Feature overview
  - scenarios: Happy path AND at least two error paths
  - out_of_scope: What won't be implemented
  - impacted_artifacts: Which code files will change
  - acceptance_criteria: How to verify it works
```

**In this experiment:**

Both A and B received the same proposal text, which already satisfied prd.schema.yaml. Both proposals listed:
- Happy path: Create and list items, then soft-delete one
- Error path 1: Delete a missing item → 404
- Error path 2: Delete with invalid UUID → 422
- Out-of-scope: Permission checks, audit log
- Impacted artifacts: service layer, router, tests, frontend
- Acceptance criteria: Tests pass, OpenAPI current, browser smoke test passes

A's agent had the scenario list but was not compelled to ensure every scenario was handled in code. B's agent was (via `feature.schema.yaml`).

### 2. feature.schema.yaml — Implementation schema

Source file: [../source/delorean/schemas/feature.schema.yaml](../source/delorean/schemas/feature.schema.yaml)

Defines a checklist of decisions that must be made and artifacts that must exist:

```yaml
required_checklist_items:
  - service_layer: "Business logic lives in services, routers stay thin"
  - typed_models: "Pydantic models for all requests/responses"
  - http_status_codes_explicit: "Route decorator declares all possible status codes"
  - tests_added: "New tests for happy path and all error scenarios"
  - error_state_handled: "Frontend shows errors and allows recovery"
  - loading_state_handled: "Frontend shows in-flight state during async ops"
  - openapi_check_passes: "make check-openapi reports the spec is current"
```

**In this experiment:**

- A's agent had access to this file but was not told to use it.
- B's agent was told to use it as a working checklist. B's compliance log is organized around these items.

This is where forced enumeration happens:

- `http_status_codes_explicit` forced B to ask: "What status codes does my DELETE route return?" → led to 404 in decorator → led to 204 convention lookup
- `loading_state_handled` forced B to ask: "How does the user know deletion is in progress?" → led to `deletingId` state in frontend
- `tests_added` forced B to ask: "Have I tested all scenarios from the proposal?" → led to 404 test, 422 test

A's agent knew these were good practices (they're common sense) but wasn't compelled to enumerate them. So A skipped loading state and the 404 test didn't verify the OpenAPI spec.

### 3. evidence.schema.yaml — Compliance receipt schema

Source file: [../source/delorean/schemas/evidence.schema.yaml](../source/delorean/schemas/evidence.schema.yaml)

Defines what must be recorded about the implementation decisions:

```yaml
evidence_summary:
  checks_run:
    - command: "What verification commands ran?"
    - outcome: "pass or fail?"
    - notes: "Why did it pass/fail?"
  tests_result:
    - passed: Number of tests
    - failed: Number of tests
    - skipped: Number of tests
  schema_compliance:
    - link_to_compliance_log: "Where is the audit trail?"
  reviewer_notes:
    - "What should a human verify before approving?"
```

**In this experiment:**

- B produced [evidence/B/compliance-log.yaml](../evidence/B/compliance-log.yaml) and [evidence/B/summary.yaml](../evidence/B/summary.yaml).
- A produced no equivalent (there was no compulsion to produce it).

B's compliance log is the key artifact: it traces every design choice back to a schema rule.

## How schemas create forced enumeration

Schemas work by turning "optional" into "required-to-document."

**Without schemas:**
- Agent: "I could add in-flight state to the Delete button, but I could also skip it."
- Decision: Skip it (less work).

**With schemas:**
- Schema says: `loading_state_handled` is a checklist item.
- Agent: "I must document whether loading state is handled. If not, why not?"
- Question: "Is loading state handled?" → Must answer → Must say why if answer is no → "No, I didn't add it" → Contradiction: but the checklist is required → Must add it.

The magic is not "the agent didn't know how to add loading state." The magic is "the agent was forced to answer the question."

Forced enumeration → Deliberate decision-making → Better outcomes.

## Key insight from this experiment

**The mechanism is not "schemas provide knowledge."**

Both A and B had the same standards directory, the same architecture docs, the same conventions. The difference is:

- **A:** Had access to knowledge but wasn't compelled to use it.
- **B:** Had a checklist that forced enumeration, which triggered deliberate consultation of knowledge.

It's the difference between:
- "You can read this book if you want." (A)
- "You must answer: what does the REST standard say about DELETE? List all the rules." (B)

The second produces better outcomes because the question forces deliberation.

## Relationship to other schema systems

### Pydantic/JSON schemas
These validate data structure (field types, required fields, constraints). Our schemas are different: they validate **decisions** (did you enumerate every status code?).

You could use Pydantic to validate that compliance-log.yaml has the right structure. But the decision schema itself is about *what decisions must be documented*, not about the structure of the documentation.

### Database schemas
These enforce data shape at the database layer. Our schemas enforce decision shape at the development layer.

### OpenAPI schemas
These document API contracts. Our schemas enforce that you *create* complete API contracts (including all status codes).

## How to read this experiment with schemas in mind

When you look at [FINDINGS.md](../FINDINGS.md), ask yourself:

1. **What decision was forced?** ("What status codes does DELETE return?")
2. **By which schema rule?** (`http_status_codes_explicit`)
3. **What artifact was created as a result?** (404 in router decorator, 404 in OpenAPI spec)

B's compliance log makes this explicit. A's code doesn't; you have to infer it.

## Further reading

- [FINDINGS.md](../FINDINGS.md) — Three divergences with schema rule traces
- [evidence/B/compliance-log.yaml](../evidence/B/compliance-log.yaml) — Audit trail showing which schema rules applied where
- [PROCESS.md](../PROCESS.md) — How we kept the comparison fair
