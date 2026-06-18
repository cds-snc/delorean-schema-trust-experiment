# What is DeLorean?

DeLorean is a framework for testing paths to AI autonomy level 5 (full autonomy with compliance and oversight).

## Canonical links

- DeLorean framework repo: https://github.com/cds-snc/delorean
- Sample project repo used for this experiment: https://github.com/cds-snc/delorean_sampleProject
- OpenSpec framework repo: https://github.com/fission-ai/openspec
- Vendored schema files in this repo: [../source/delorean/schemas/](../source/delorean/schemas/)

## High-level purpose

DeLorean measures what happens when you add a **constraint layer** to an autonomous agent's workflow:

- **Without constraint layer:** Agent has access to standards, architecture docs, and best practices, but is not compelled to use them systematically. Gaps may go undetected.
- **With constraint layer:** Agent must enumerate decisions against explicit schema rules and produce compliance receipts. Gaps are harder to hide.

This experiment tests the "with constraint layer" pathway by comparing two otherwise-identical agent runs on the same feature.

## What DeLorean adds (not the OpenSpec framework, but the layer on top)

DeLorean's constraint layer consists of:

### 1. Typed field schemas (YAML)

- `prd.schema.yaml` — Requires every feature proposal to list error scenarios and out-of-scope boundaries ([sample path](https://github.com/cds-snc/delorean_sampleProject/tree/main/delorean/schemas))
- `feature.schema.yaml` — Checklist of implementation requirements (service layer, typed models, explicit status codes, tests for error paths, etc.) ([sample path](https://github.com/cds-snc/delorean_sampleProject/tree/main/delorean/schemas))
- `evidence.schema.yaml` — Structure for compliance receipts (what checks ran, did they pass, notes) ([sample path](https://github.com/cds-snc/delorean_sampleProject/tree/main/delorean/schemas))

### 2. Compliance logs

- `compliance-log.yaml` — Audit trail: each schema rule that was applied, what it required, and what changed as a result
- `summary.yaml` — Evidence summary: test results, linting, OpenAPI freshness, reviewer notes

### 3. Gates (decision points)

Work-context gates, control-boundary gates, approval/waiver gates — not all tested in this experiment, but available in the full framework.

### 4. State machine

Features move through: spec → plan → implement → verify → release-ready → closed. This experiment captures the implement → verify transition.

## Why this matters for autonomy

At autonomy level 5, the agent operates unsupervised but must produce receipts that let humans audit every decision. DeLorean's constraint layer is designed to make auditing possible:

- **Compliance logs** let humans answer: "Why did the agent return 204 instead of 200?" Trace: feature schema rule → checklist item → decision.
- **Typed schemas** prevent accidental omissions (e.g., forgetting to test error paths).
- **Forced enumeration** creates a deliberate question-and-answer cycle instead of implicit defaults.

This experiment demonstrates that the constraint layer *works*: forced enumeration catches gaps that access alone misses.

## Relationship to OpenSpec

**OpenSpec** (from Fission-AI, 55.5k GitHub stars) is an upstream framework for spec/plan/task artifacts with an artifact dependency graph. DeLorean does not replace it.

DeLorean sits on top of OpenSpec:

- **OpenSpec provides:** Artifact shape (proposal.md, design.md, tasks.md), artifact dependency graph, `/opsx:verify` (3-dimensional validation)
- **DeLorean provides:** Field-level validation within those artifacts, enforcement of schema rules, compliance receipts, gates, state machine

You could think of it as:
- OpenSpec = "the shape and flow of artifacts"
- DeLorean = "the rules for what goes inside those artifacts and the proof you followed the rules"

## What DeLorean does NOT do

- **Not a full governance framework.** No approval workflows, risk scoring, or financial accountability (yet).
- **Not a deployment tool.** No CD/CI integration or release orchestration (though gates interface with it).
- **Not a database of standards.** Standards are stored separately (in `architecture_docs/`) ([sample standards path](https://github.com/cds-snc/delorean_sampleProject/tree/main/architecture_docs/standards)); DeLorean just enforces that you follow them.

## How to read this experiment with DeLorean context

When you look at [FINDINGS.md](../FINDINGS.md), you'll see three divergences between A and B:

1. **404 gap in OpenAPI** — Caused by schema rule `http_status_codes_explicit` forcing enumeration
2. **200 vs 204** — Caused by same rule forcing a lookup of REST standards
3. **Missing loading state** — Caused by schema rule `loading_state_handled` forcing deliberation

All three are examples of forced enumeration working: agents do what they're compelled to enumerate, not just what they have access to.

B's [compliance-log.yaml](../evidence/B/compliance-log.yaml) shows the audit trail: you can trace each decision back to the schema rule that prompted it. A has no compliance log, so you can only infer decisions from code inspection.

## Further reading

- [FINDINGS.md](../FINDINGS.md) — Three divergences with mechanism traces
- [PROCESS.md](../PROCESS.md) — How the experiment was kept fair
- [evidence/B/compliance-log.yaml](../evidence/B/compliance-log.yaml) — Full audit trail from B's run
- [evidence/B/summary.yaml](../evidence/B/summary.yaml) — B's evidence summary
