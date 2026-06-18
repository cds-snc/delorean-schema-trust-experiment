# Schema Trust A/B — Comparison

**Date:** 2026-06-18

## Setup

Two worktrees, two purpose-built baselines so each track's worktree only
contains the materials that track is allowed to see:

|                                      | A (unguided)                                                                                                                                                      | B (schema-guided)            |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| Branch                               | `experiment/a-unguided`                                                                                                                                           | `experiment/b-schema-guided` |
| Baseline SHA                         | `0f17849`                                                                                                                                                         | `ff119dc`                    |
| `delorean/schemas/`                  | absent                                                                                                                                                            | present                      |
| `AGENTS.md` Schema Contracts section | absent                                                                                                                                                            | present                      |
| Both agents                          | same proposal, same model (Claude Opus 4.7, high reasoning, 200K), same agent mode, brand-new chat session, sequential runs against the same DB on the same ports |                              |

Both features were UI-smoke-tested in the browser after the run. **Both
work end-to-end.** Both soft-delete in PostgreSQL. Both have a
`window.confirm` dialog. All claimed tests pass.

---

## Headline metrics

| Metric                    | A (unguided)      | B (schema-guided)                      | Δ (B vs A)       |
| ------------------------- | -----------------:| --------------------------------------:| ----------------:|
| Chat session id           | `90976e56-…756b3` | `dd56b5cf-…cc06e01`                    |                  |
| Turns                     | 58                | 44                                     | −14 (−24%)       |
| Tool calls                | 57                | 74                                     | +17 (+30%)       |
| Tools per turn            | 0.98              | 1.68                                   | +71%             |
| Wall-clock (transcript)   | 5.16 min          | 6.10 min                               | +56 s (+18%)     |
| Credits (session, UI)     | 248.8             | 257.1                                  | +8.3 (+3.3%)     |
| Approvals (user estimate) | 5–6               | 9–12                                   | ~2×              |
| Files changed             | 9                 | 11                                     | +2               |
| Lines added               | 287               | 433                                    | +146             |
| Lines deleted             | 3                 | 22                                     | +19              |
| Tests passing             | 11                | 12                                     | +1               |
| Coverage                  | 90%               | 92%                                    | +2 pp            |
| Schema-audit artifacts    | none              | `compliance-log.yaml` + `summary.yaml` | full audit trail |

---

## Tool breakdown

| Tool                   | A      | B      |
| ---------------------- | ------:| ------:|
| read_file              | 21     | 27     |
| list_dir               | 15     | 19     |
| replace_string_in_file | 10     | 14     |
| manage_todo_list       | 5      | 1      |
| run_in_terminal        | 3      | 8      |
| grep_search            | 2      | 2      |
| create_file            | 1      | 2      |
| file_search            | 0      | 1      |
| **total**              | **57** | **74** |

A leaned on a manual TODO list to stay organized (5 `manage_todo_list`
calls). B leaned on the schema checklist + terminal verification (1 todo
call, 8 terminal calls). B used the schema as its plan; A invented its
own.

---

## What the side-by-side code reveals

Both implementations are architecturally similar: both put the delete
business logic in `ExampleItemService`, both add typed
`get_active_by_public_id` + `soft_delete` methods to the repository, both
use SQLAlchemy ORM (no raw SQL), both wire a `window.confirm` dialog.

Three real divergences, all driven by the same mechanism: **schemas
force enumeration.** Sometimes that produces an artifact directly
(the 404 in B's OpenAPI spec). Sometimes it produces the question that
sends the agent to a standard it already had access to (204 from
std-009). Either way the rule is the same — agents do what they're
compelled to enumerate, not what they merely have access to.

### 1. OpenAPI contract completeness

|                                                             | A            | B                 |
| ----------------------------------------------------------- | ------------ | ----------------- |
| `DELETE /v1/example-items/{public_id}` documented responses | `{200, 422}` | `{204, 404, 422}` |

**A's router raises `HTTPException(404)` for missing items, but A's
OpenAPI spec does not document the 404 response.** B's does.

Concrete consequences:

- Any SDK / typed client generated from A's spec will not know to
  distinguish "not found" from a network error.
- A's tests pass (the route still raises 404; tests don't verify the
  spec).
- The gap is invisible to lint, type-checking, and the unit suite. It
  only surfaces when an external consumer asks "where's the 404
  handling?" — usually weeks or months later.

This is the most consequential difference between the two outputs.

**Why B caught it (schemas as checklists):** A three-link chain across
the schema corpus. `prd.schema.yaml` requires every proposal's scenarios
to cover an error path → B's proposal listed "delete a missing item" →
`feature.schema.yaml` requires `http_status_codes_explicit` + an
error-path test → 404 ends up in the route decorator + the test suite →
`feature.schema.yaml` requires `openapi_check_passes` as the final
confirmation → 404 ends up in `openapi/openapi.json`. The bug never
becomes possible.

# 

### 2. HTTP response semantics

|                               | A                                        | B                                          |
| ----------------------------- | ---------------------------------------- | ------------------------------------------ |
| Service `delete_item` returns | `ExampleItemRead` (the deleted item)     | `None`                                     |
| Router                        | `response_model=ExampleItemResponse`     | `status_code=204, response_class=Response` |
| Wire response                 | `200 OK` + JSON body of the deleted item | `204 No Content`, empty body               |

Both internally consistent (A's frontend parses the JSON it gets back).
Functional impact is small for this app. External impact is real:

- HTTP intermediaries (caches, proxies, CDNs) treat `200 + body` and
  `204` differently. A's pattern surprises any standard tooling.
- A `204 No Content` is the REST convention for a successful destructive
  operation. A's `200 + body` is non-idiomatic and will confuse any
  external consumer reading the API.

**Why B caught it (forced enumeration):** Both worktrees ship the
identical `architecture_docs/standards/std-009-api-rest.md`, which
prescribes `204` for `DELETE`. Both `AGENTS.md` files contain the same
pointer to the standards corpus. The difference is that B's
`feature.schema.yaml` requires `http_status_codes_explicit` — the agent
is compelled to declare every status code it returns. That requirement
turns "pick a status code" into a question that has to be answered
deliberately, which sends the agent to std-009 to look up the
convention. A was never compelled to enumerate, so A reached for the
first pattern that worked (return the deleted object, get 200) and
never opened std-009. B's compliance log credits the schema rule
explicitly: *"http_status_codes_explicit required declaring
status_code=204 and the 404 response shape."*

### 

### 3. UX in-flight state

|                                      | A    | B                                                                      |
| ------------------------------------ | ---- | ---------------------------------------------------------------------- |
| `ExampleDataPage.tsx` deleting state | none | `deletingId` tracked, button disabled, label switches to `"Deleting…"` |

User impact: double-clicking Delete in A fires two DELETE requests
back-to-back. The second 404s; the UI doesn't show it. B prevents the
double-click at the source.

A 4-line oversight from the same proposal both agents read.

---

## Test names as contract documentation

Same coverage, different naming convention. B's names describe the
contract; A's describe the behavior. To a reviewer, B's test list reads
like an API spec; A's reads like a TODO list.

| A                                                         | B                                                       |
| --------------------------------------------------------- | ------------------------------------------------------- |
| `test_example_items_route_deletes_existing_item`          | `test_example_items_delete_route_returns_204`           |
| `test_example_items_route_maps_missing_item_to_not_found` | `test_example_items_delete_route_maps_not_found_to_404` |
| *(none — A has no 422 test)*                              | `test_example_items_delete_route_rejects_invalid_uuid`  |

---

## Conclusion

**Schemas don't make the work better — they make the work defensible.**

For a capable model, schemas:

1. **Enforce contract completeness.** B's OpenAPI spec documents every
   status code the route can return. A's silently omits the 404 it
   actually raises. This is the bug type schemas reliably catch and
   nothing else does.

2. **Produce an audit trail.** B's `compliance-log.yaml` ties every
   implementation decision to a specific schema field. A produces working
   code with no equivalent record. At review time, B's choices are
   traceable; A's are inferred from the diff.

3. **Enforce REST and UX conventions the model would otherwise
   improvise.** A's `200 + body` DELETE works for A's own UI but
   surprises any other consumer. A's missing in-flight state ships a
   double-click hazard. Both individually small. Both the kind of thing
   schemas catch reliably.

What schemas do not do:

- Prevent bugs a capable model would not write. A built the same service
  split, the same typed repo, the same confirm dialog without being
  told.
- Justify themselves on credit cost. The 3.3% credit premium is
  irrelevant — the question is whether you can hand the output to a
  team without re-reading every line.

---

## Files in this archive

```
run-2/
├── COMPARISON.md           (this file)
├── A/
│   ├── git-diff-stat.txt
│   ├── git-status.txt
│   ├── pytest-output.txt
│   └── manual-metrics.txt
└── B/
    ├── git-diff-stat.txt
    ├── git-status.txt
    ├── pytest-output.txt
    ├── compliance-log.yaml     ← schema audit trail
    ├── summary.yaml            ← evidence summary
    └── manual-metrics.txt
```

Raw harvest (transcripts, full diffs, both evidence trees) is preserved
at `experiment-logs/run-2-20260618-112733/`. VS Code workspace storage
for both chat sessions is intact.
