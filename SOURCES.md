# Source Provenance Manifest

This repository contains source-backed artifacts for the schema trust A/B experiment.

## Upstream repositories

- DeLorean framework: https://github.com/cds-snc/delorean
- Sample project: https://github.com/cds-snc/delorean_sampleProject

## Copy map

### Experiment evidence (raw)

Copied from:
- /Users/jesseburcsik/Documents/dev_projects/delorian/delorean_sampleProject/delorean/evidence/schema-trust-ab/run-2/A/
- /Users/jesseburcsik/Documents/dev_projects/delorian/delorean_sampleProject/delorean/evidence/schema-trust-ab/run-2/B/

Included files:
- evidence/A/git-diff-stat.txt
- evidence/A/manual-metrics.txt
- evidence/A/pytest-output.txt
- evidence/B/git-diff-stat.txt
- evidence/B/manual-metrics.txt
- evidence/B/pytest-output.txt
- evidence/B/compliance-log.yaml
- evidence/B/summary.yaml

### Runbook documents (raw)

Copied from:
- /Users/jesseburcsik/Documents/dev_projects/delorian/delorean_sampleProject/delorean/evidence/schema-trust-ab/run-2/COMPARISON.md
- /Users/jesseburcsik/Documents/dev_projects/delorian/delorean_sampleProject/delorean/evidence/schema-trust-ab/run-2/HOW-WE-RAN-IT.md

Included files:
- inputs/runbooks/COMPARISON.md
- inputs/runbooks/HOW-WE-RAN-IT.md

### Proposal inputs (raw)

Copied from:
- /Users/jesseburcsik/Documents/dev_projects/delorian/delorean_sampleProject_A/openspec/changes/delete-unguided/proposal.md
- /Users/jesseburcsik/Documents/dev_projects/delorian/delorean_sampleProject_B/openspec/changes/delete-schema-guided/proposal.md

Included files:
- inputs/proposals/A/proposal.md
- inputs/proposals/B/proposal.md

### Schema sources (raw)

Copied from:
- /Users/jesseburcsik/Documents/dev_projects/delorian/delorean_sampleProject/delorean/schemas/

Included files:
- source/delorean/schemas/compliance-log.schema.yaml
- source/delorean/schemas/evidence.schema.yaml
- source/delorean/schemas/feature.schema.yaml
- source/delorean/schemas/feature.schema.yaml.md
- source/delorean/schemas/prd.schema.yaml

## Prompt evidence status

Exact A/B run-session prompt text is not currently included as a verified verbatim source artifact in this package.

- Reconstructed prompt text files were removed.
- PROCESS.md still contains prompt wording from runbook/process notes.
- If verbatim prompt logs are recovered later, add them under inputs/prompts/ with source path and capture timestamp.

## Integrity rule

Files in this package should be either:
- direct copies of source artifacts, or
- provenance/index metadata describing those artifacts.

Interpretive analysis files are allowed, but must be clearly separate from raw evidence.
