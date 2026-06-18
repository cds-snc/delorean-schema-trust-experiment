# Schema Trust: An A/B Experiment

**What we tested:** Does schema-guided development produce better code than unguided development, if both use the same model, same feature spec, and same codebase?

**The answer:** Yes. And more specifically: schemas force enumeration, which catches gaps that access alone doesn't reach.

---

## Quick navigation

**New here?**
- Start with [FINDINGS.md](FINDINGS.md) — three headline divergences with concrete examples.
- Then read [PROCESS.md](PROCESS.md) — how we kept it fair.
- Dive into [evidence/](evidence/) for the raw artifacts.

**Want context?**
- [What is DeLorean?](understand/what-is-delorean.md) — the autonomy framework this experiment measures.
- [What are schemas?](understand/what-are-schemas.md) — the constraint layer we added.
- [Why is this fair?](understand/fair-comparison.md) — how we isolated the variable.
- [How to read the findings](understand/how-to-read-findings.md) — guide to observability.

**Want the raw data?**
- [evidence/A/](evidence/A/) — Track A's metrics and output (unguided).
- [evidence/B/](evidence/B/) — Track B's metrics and output (schema-guided), plus compliance log.
- [evidence/comparison/](evidence/comparison/) — Side-by-side analysis of each divergence.

---

## The experiment at a glance

| | **Track A (unguided)** | **Track B (schema-guided)** |
|---|---|---|
| **Starting point** | Fresh baseline: no schemas, no schema-audit section in AGENTS.md | Fresh baseline: schemas present, schema-audit section in AGENTS.md |
| **Feature** | Soft-delete with 404 handling on a REST DELETE endpoint | Same feature |
| **Model** | Claude Opus 4.7, high reasoning, 200K context | Same |
| **Input** | Proposal: feature description + standard design template | Proposal + one paragraph: "use the schema-trust standard" |
| **Fairness** | Both worktrees isolated on separate branches; no cross-contamination | Same environment, same database, sequential runs |

**Result:** Both features work end-to-end. Three code differences, all traceable to one mechanism.

---

## The three divergences

(Read [FINDINGS.md](FINDINGS.md) for detailed analysis with mechanism traces.)

### 1. OpenAPI contract completeness
- **A's DELETE route:** Documents responses `[200, 422]`; raises `404` but doesn't document it.
- **B's DELETE route:** Documents responses `[204, 404, 422]`; contract is complete.
- **Impact:** A's consumers can't distinguish "not found" from network errors.

### 2. HTTP DELETE semantics
- **A returns:** `200 OK + {deleted item JSON}`
- **B returns:** `204 No Content + empty`
- **Standard:** REST API convention (STD-009) prescribes `204` for successful destructive operations.
- **Impact:** A violates REST idiom; external consumers will be confused.

### 3. UX in-flight state
- **A's Delete button:** No state tracking; double-clicking fires two DELETE requests.
- **B's Delete button:** Tracks `deletingId` state, disables button, shows "Deleting…".
- **Impact:** A allows accidental duplicate deletes.

---

## Why this matters

**Same model, same feature, same codebase.** The only difference is whether the agent was compelled to enumerate decisions against explicit rules.

When you enumerate, you document what you're doing and why. That creates three kinds of artifacts:

1. **Compliance receipts** (B's `compliance-log.yaml`) — a trail of which schema rules applied where.
2. **Complete contracts** (B's OpenAPI spec with all status codes) — forced completeness.
3. **Deliberate choices** (B's 204 instead of A's 200) — because the rule forced a lookup.

This is not "B had better access to knowledge." Both baselines had the same standards directory. The difference is **B was compelled to use it.**

---

## The mechanism: forced enumeration

Schemas work by turning "nice to have" into "must document." 

- A had access to STD-009 (REST API) but wasn't compelled to read it.
- B's `feature.schema.yaml` rule `http_status_codes_explicit` required the agent to enumerate every status code.
- Enumeration required answering: "What status codes does DELETE return?"
- That question led to STD-009 → `204 No Content`.

Same model, same knowledge, different compulsion level.

---

## Files in this repo

```
evidence/
├── A/                           # Track A artifacts
│   ├── git-diff-stat.txt        # What changed (9 files, 287 lines added)
│   ├── manual-metrics.txt       # Chat turns, credits, approvals
│   └── pytest-output.txt        # Test results (11 passed, 90% coverage)
├── B/                           # Track B artifacts
│   ├── git-diff-stat.txt        # What changed (11 files, 433 lines added)
│   ├── manual-metrics.txt       # Chat turns, credits, approvals
│   ├── pytest-output.txt        # Test results (12 passed, 92% coverage)
│   ├── compliance-log.yaml      # Schema audit trail (10 events)
│   └── summary.yaml             # Evidence summary with reviewer notes
└── comparison/
    ├── divergence-1-openapi.md  # 404 gap trace
    ├── divergence-2-delete-semantics.md  # 200 vs 204
    └── divergence-3-ux-state.md # In-flight state

understand/
├── what-is-delorean.md          # Context: DeLorean autonomy framework
├── what-are-schemas.md          # Context: constraint layer added
├── fair-comparison.md           # Why A vs B is fair
└── how-to-read-findings.md      # Guide to observability

FINDINGS.md                       # Three divergences with traces
PROCESS.md                        # How we ran it
LICENSE
```

---

## Key takeaways

1. **Schemas force enumeration.** Agents do what they're compelled to enumerate, not just what they have access to.
2. **Enumeration produces artifacts.** Compliance logs, complete contracts, and deliberate choices all flow from forced enumeration.
3. **The mechanism is two-path:** Direct artifact generation (404 in decorator) or forced consultation (looking up std-009 to answer a required question).
4. **This is not about access.** Both baselines had identical standards directories and identical AGENTS.md pointers. Only B was compelled to use them.

---

## Authorship & license

Experiment designed and executed in June 2026 as part of research into schema-guided autonomous development.

See [LICENSE](LICENSE) for terms.
