# What is DeLorean?

DeLorean is an existing shared project and way of working for teams experimenting with higher-trust agentic software delivery.

It includes operating guidance, repo structure, prompts, skills, evidence patterns, local process records, and supporting templates used across solution repositories. This repository documents one additional experiment built on top of that broader project; it does not define DeLorean as a whole.

## Canonical links

- DeLorean framework repo: https://github.com/cds-snc/delorean
- Sample project repo used for this experiment: https://github.com/cds-snc/delorean_sampleProject
- OpenSpec framework repo: https://github.com/fission-ai/openspec
- Vendored schema files in this repo: [../source/delorean/schemas/](../source/delorean/schemas/)

## High-level purpose

At a high level, DeLorean is aimed at making agent-assisted and agent-driven delivery more reviewable, safer, and easier to operate in real solution repositories.

For this particular experiment, we added a **schema-driven constraint layer** on top of the existing DeLorean project:

- **Without constraint layer:** Agent has access to standards, architecture docs, and best practices, but is not compelled to use them systematically. Gaps may go undetected.
- **With constraint layer:** Agent must enumerate decisions against explicit schema rules and produce compliance receipts. Gaps are harder to hide.

This experiment tests that added layer by comparing two otherwise-identical agent runs on the same feature.

## What belongs to DeLorean vs. this experiment

In the sample project, DeLorean already includes more than schemas:

- local process records under `delorean/`
- evidence and approval/waiver patterns
- gates and adoption-level behavior
- prompts, skills, hooks, and repo guidance used by teams working in the project
- integration with shared architecture guidance and OpenSpec workflows

What this experiment adds on top is a narrower constraint mechanism centered on schemas and compliance receipts.

## What the experiment adds on top

The added constraint layer in this experiment consists primarily of:

### 1. Typed field schemas (YAML)

- `prd.schema.yaml` — Requires every feature proposal to list error scenarios and out-of-scope boundaries ([sample path](https://github.com/cds-snc/delorean_sampleProject/tree/main/delorean/schemas))
- `feature.schema.yaml` — Checklist of implementation requirements (service layer, typed models, explicit status codes, tests for error paths, etc.) ([sample path](https://github.com/cds-snc/delorean_sampleProject/tree/main/delorean/schemas))
- `evidence.schema.yaml` — Structure for compliance receipts (what checks ran, did they pass, notes) ([sample path](https://github.com/cds-snc/delorean_sampleProject/tree/main/delorean/schemas))

### 2. Compliance logs

- `compliance-log.yaml` — Audit trail: each schema rule that was applied, what it required, and what changed as a result
- `summary.yaml` — Evidence summary: test results, linting, OpenAPI freshness, reviewer notes

## Why this matters for this experiment

For this experiment, the added schema layer makes it easier to audit specific implementation decisions:

- **Compliance logs** let humans answer: "Why did the agent return 204 instead of 200?" Trace: feature schema rule → checklist item → decision.
- **Typed schemas** prevent accidental omissions (e.g., forgetting to test error paths).
- **Forced enumeration** creates a deliberate question-and-answer cycle instead of implicit defaults.

This experiment is intended to test whether that added layer catches gaps that access alone misses.

## Relationship to OpenSpec

**OpenSpec** (from Fission-AI, 55.5k GitHub stars) is an upstream framework for spec/plan/task artifacts with an artifact dependency graph. DeLorean does not replace it.

DeLorean sits on top of OpenSpec:

- **OpenSpec provides:** Artifact shape (proposal.md, design.md, tasks.md), artifact dependency graph, `/opsx:verify` (3-dimensional validation)
- **DeLorean provides more broadly:** operating guidance, local process structure, evidence patterns, gates, and repo-level workflows around those artifacts
- **This experiment adds:** field-level schema validation, compliance receipts, and explicit enumeration checks as one testable layer on top

You could think of it as:
- OpenSpec = "the shape and flow of artifacts"
- DeLorean = "the broader operating model and project structure around agent-assisted delivery"
- This experiment = "one added schema-driven control layer tested within that broader model"

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
