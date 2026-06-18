# Process: How We Ran the Experiment

This document explains how we isolated the variable and kept the comparison fair.

---

## The question

If we ask an AI agent to build the same feature twice — once with our schema-trust standard, once without — does it actually matter?

Base repositories used in this experiment:
- DeLorean framework: https://github.com/cds-snc/delorean
- Sample project: https://github.com/cds-snc/delorean_sampleProject

---

## The setup: One feature, two agents, one variable

Canonical input artifacts in this repo:
- Prompts: [inputs/prompts/](inputs/prompts/)
- Proposal used for Track A: [inputs/proposals/A/proposal.md](inputs/proposals/A/proposal.md)
- Proposal used for Track B: [inputs/proposals/B/proposal.md](inputs/proposals/B/proposal.md)

### What was the same

| Aspect | Value |
|--------|-------|
| **Feature** | Soft-delete an item with 404 error handling on REST DELETE endpoint |
| **Proposal** | Feature description + standard design template (identical for both tracks) |
| **Model** | Claude Opus 4.7 (released 2025-12-19) |
| **Model settings** | High reasoning enabled, 200K context window |
| **Agent mode** | Chat agent mode in VS Code |
| **Environment** | PostgreSQL database, FastAPI backend, React frontend |
| **Database** | Shared instance, sequential runs (A first, then B) |
| **Ports** | Backend 8000/8001, Frontend 3000/3001 |
| **Chat session** | Brand new session for each track (no history carry-over) |
| **Fairness checkpoint** | Both agents saw identical proposal text; only AGENTS.md differed |

### What was different

| Aspect | Track A (unguided) | Track B (schema-guided) |
|--------|---|---|
| **Branch** | `experiment/a-unguided` | `experiment/b-schema-guided` |
| **Baseline commit** | `0f17849` | `ff119dc` |
| **`delorean/schemas/` directory** | Absent (removed from baseline) | Present (with `prd.schema.yaml`, `feature.schema.yaml`, `evidence.schema.yaml`) |
| **`AGENTS.md` section** | No "Schema Contracts" section (removed from baseline) | "Schema Contracts" section present, describes schema-audit workflow |
| **Proposal requirement** | Standard feature spec | Feature spec + one paragraph: "Please use the schema-trust standard for this feature" |

### Isolation: Why separate baselines matter

Both branches forked from their own baseline commit. This prevented cross-contamination:

- A's baseline (`0f17849`) does not include schemas, so A's agent never had the option to use them.
- B's baseline (`ff119dc`) includes schemas, so B's agent had the option and the guidance to use them.

If we had used a single branch and removed schemas for A, the agent would have seen the removal history and might have inferred "schemas were removed because they don't matter." By using separate branches with isolated baselines, each track only saw the materials it was meant to see.

---

## Fairness controls

### Same starting point, isolated paths

1. **Codebase snapshot:** Both baselines were taken on the same day from a common ancestor. No algorithmic differences in the app structure, only the presence/absence of schemas and their mentions in AGENTS.md.

2. **No cross-branch bleeding:** Each track ran in a separate VS Code window with a separate git worktree. No accidental file sharing or copy-paste errors.

3. **Sequential database runs:** Both hit the same PostgreSQL instance sequentially (A first, then B after A's transactions committed). This ensures identical starting data and no interference.

4. **Proposal identical except one sentence:** Both agents received the same feature description. B received one additional sentence: "Please follow the schema-trust standard, including schemas in `delorean/schemas/`, and create a compliance log in `delorean/evidence/` that traces each design choice back to schema requirements."

---

## What we measured

### Throughput metrics

Captured from the Claude chat UI and manual timing:

| Metric | Unit |
|--------|------|
| **Chat turns** | Count of user messages + agent responses |
| **Tool calls** | Total (read_file, run_in_terminal, etc.) |
| **Wall-clock time** | Elapsed seconds from start to finish |
| **Model credits** | Shown in chat session UI after completion |
| **User approvals** | Estimated count of "go ahead" confirmations |

### Output metrics

Captured from git after each run:

| Metric | Data source |
|--------|-------------|
| **Files changed** | `git diff --stat` between baseline and run commit |
| **Lines added/deleted** | Same as above |
| **Tests passing** | `pytest` output at end of run |
| **Code coverage** | Coverage report from pytest run |

### Behavioral artifacts

Captured by running both implementations end-to-end:

| Artifact | How captured |
|----------|-------------|
| **OpenAPI spec** | `make export-openapi` after each run; compared responses documented |
| **DELETE status codes** | Hit DELETE endpoint with valid ID, missing ID, invalid UUID; recorded responses |
| **Frontend state** | Clicked Delete button in browser, observed button state and loading label |
| **Test file contents** | Compared test names and assertions between A and B |

### What we did NOT measure

- **Subjective code quality:** We did not ask human reviewers to rate A vs B. Both features work.
- **Long-term maintainability:** One afternoon of work cannot speak to drift over months.
- **Generalization:** This is one feature, one model, one moment. Treat as a signal, not proof.
- **Agent iteration count:** We did not compare how many times the agent had to revise code (both were single-pass, so this was equivalent).

---

## How each track was run

### Track A (unguided)

1. **Checkout:** `git checkout experiment/a-unguided`
2. **Start:** Fresh VS Code window, new Claude chat session
3. **Prompt:** "Please implement the soft-delete feature as described in the proposal. Use the codebase conventions and architectural standards."
4. **Agent behavior:** Invented its own plan (used `manage_todo_list` 5 times for self-organization); consulted standards as needed but was not compelled to be exhaustive
5. **Completion:** Feature builds, tests pass (11 passing), browser smoke test passes
6. **Metrics captured:** Turn count, credits, approvals; git diff stat; pytest output; OpenAPI spec
7. **Evidence archived:** Stored in [evidence/A/](evidence/A/)

### Track B (schema-guided)

1. **Checkout:** `git checkout experiment/b-schema-guided`
2. **Start:** Fresh VS Code window, new Claude chat session
3. **Prompt:** Same feature description + "Please use the schema-trust standard. Apply the checklist in `delorean/schemas/feature.schema.yaml`. Create a compliance log in `delorean/evidence/` that documents each decision against the schema requirements."
4. **Agent behavior:** Used the schema as its plan (1 `manage_todo_list` call); leaned on terminal verification to confirm compliance; enumerated design choices deliberately
5. **Completion:** Feature builds, tests pass (12 passing), browser smoke test passes
6. **Metrics captured:** Same as A, plus `compliance-log.yaml` and `summary.yaml` from B's run
7. **Evidence archived:** Stored in [evidence/B/](evidence/B/)

---

## Key metrics

### Throughput comparison

| Metric | A | B | Δ |
|--------|---|---|---|
| **Turns** | 58 | 44 | −14 (−24%) |
| **Tool calls** | 57 | 74 | +17 (+30%) |
| **Tools per turn** | 0.98 | 1.68 | +71% |
| **Wall-clock (min)** | 5.16 | 6.10 | +54 s (+18%) |
| **Credits** | 248.8 | 257.1 | +8.3 (+3.3%) |
| **Approvals** | 5–6 | 9–12 | ~2× |

**Interpretation:**
- A used more turns but fewer tools per turn (more conversational, more back-and-forth).
- B used fewer turns but more tools per turn (more deliberate, each turn accomplished more).
- A took less wall-clock time but required more user approvals (asked "is this OK?" more often).
- B took slightly more time but automated checks and verification reduced approval requests.

### Output comparison

| Metric | A | B | Δ |
|--------|---|---|---|
| **Files changed** | 9 | 11 | +2 |
| **Lines added** | 287 | 433 | +146 |
| **Lines deleted** | 3 | 22 | +19 |
| **Tests passing** | 11 | 12 | +1 |
| **Coverage** | 90% | 92% | +2 pp |

**Interpretation:**
- B made more targeted changes (more deletions, more structured additions).
- B added more tests (specifically for 404 and 422 edge cases).
- B's coverage is 2 percentage points higher.

---

## Fair comparison checklist

✓ **Same model and settings** — Claude Opus 4.7, high reasoning, 200K context, agent mode  
✓ **Same feature spec** — Identical proposal text  
✓ **Same codebase baseline** — Both from same ancestor, diverged only on schema presence  
✓ **Isolated paths** — Separate git branches, separate VS Code windows  
✓ **Same environment** — Shared database, same ports, sequential runs  
✓ **Brand new sessions** — No chat history carry-over between tracks  
✓ **One variable changed** — Presence/absence of schemas + guidance  
✓ **Full capture** — Every turn, tool call, and file change recorded  

This is as fair a 1:1 comparison as we could design without running the agents side-by-side in the same session (which would introduce interference).

---

## Limitations and caveats

1. **Sample size of one:** One feature, one moment. Signals are not proofs.
2. **Model-specific:** Opus 4.7 results may not generalize to other models.
3. **Shallow feature:** Soft-delete is straightforward; schema benefits might be smaller or larger for complex features.
4. **No long-term data:** We can't measure drift, refactoring, or handoff over weeks/months.
5. **No code review:** We didn't ask humans to rate either output; both are valid implementations.
6. **Sequential, not parallel:** Running B after A means B's agent could theoretically see side effects from A (though database was reset). A true parallel run would require two synchronized environments.

---

## Next steps

- Read [FINDINGS.md](FINDINGS.md) for the three divergences and mechanism traces.
- Inspect [evidence/B/compliance-log.yaml](evidence/B/compliance-log.yaml) to see how schema enumeration works in practice.
- Compare [evidence/A/](evidence/A/) and [evidence/B/](evidence/B/) raw metrics side-by-side.
