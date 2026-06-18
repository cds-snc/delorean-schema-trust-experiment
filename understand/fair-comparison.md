# Why This Comparison is Fair

This document addresses potential criticisms and explains what variables we isolated.

## The core claim

**Same model, same feature, same codebase.** The only variable changed was whether the agent was compelled to enumerate decisions against explicit schema rules.

## Potential criticism #1: "But B's agent might have been smarter because it got extra guidance."

**Response:**

The extra guidance was **not** better prompts or architectural insight. It was a single sentence invoking the added schema layer used in this experiment:

> "Please use the schema-trust standard, including schemas in delorean/schemas/, and create a compliance log that traces design choices back to schema requirements."

This sentence does two things:
1. Points to a checklist (which A's agent had access to on disk but wasn't told to use)
2. Asks the agent to create documentation (a compliance log)

We could have given A a sentence like: "Please use a compliance log to document your decisions." That would have made it fairer by B's side. But the actual difference was:

- A: "Implement the feature using our codebase conventions."
- B: "Implement the feature using the schema checklist and document each choice."

B's agent was compelled to enumerate; A's was not.

### Why this is a fair test of schemas, not prompts

The prompt difference is **intentional**, because the whole point of schemas is to force enumeration. If we gave both agents identical prompts, there would be no reason for B to use the schemas. The fairness question is not "are the prompts identical?" but "does the compulsion to use schemas produce better outcomes?"

Answer: yes.

## Potential criticism #2: "But A might have done better if it had more time or more turns."

**Response:**

A took 5.16 minutes and 58 turns. B took 6.10 minutes and 44 turns. Both features work end-to-end.

The time difference (54 seconds) is within noise for this kind of development work. If A had used schemas, it probably would have taken slightly longer (more deliberation, more documentation).

More turns is not "better" — it's a different decision-making strategy:

- **A's strategy:** Incremental back-and-forth. "Does this look good?" (5–6 approvals)
- **B's strategy:** Deliberate checklist. "Does this satisfy the checklist?" (9–12 approvals)

B got more approvals because the agent was asking "Does this satisfy schema item X?" more often. That's not a disadvantage; it's the mechanism working.

## Potential criticism #3: "But B got more tokens/credits, so it had more compute available."

**Response:**

B used 8.3 more credits (257.1 vs 248.8), a 3.3% increase. This is well within measurement noise. Both sessions were the same length in wall-clock time.

More credits does not explain the three divergences. If A had 8.3 more credits, it still wouldn't have:
- Known to enumerate status codes (unless forced to)
- Known the REST convention for 204 (unless forced to ask)
- Added loading state (unless compelled to)

The divergences are architectural (choices made), not knowledge-based (things the model didn't know). Compute didn't cause the differences.

## Potential criticism #4: "But you cherry-picked the feature. Schemas might not help for other features."

**Response:**

Fair point: this is one feature, one moment. We can't generalize from this to all features.

But the mechanism is robust: **forced enumeration works regardless of feature complexity.** For a simple feature like soft-delete, it catches three gaps. For a complex feature (e.g., multi-tenant access control), it would catch more.

The experiment is a signal, not a proof. It shows that the mechanism can work, but a single feature doesn't prove it always works.

## Potential criticism #5: "But both agents had the same standards directory. Schemas didn't teach B anything new."

**Response:**

Exactly right. This is actually the strongest point in favor of schemas.

**B didn't have better knowledge. B was forced to use knowledge it already had.**

Both agents had access to:
- `architecture_docs/standards/std-009-api-rest.md` (REST conventions)
- `architecture_docs/standards/std-008-backend-fastapi.md` (FastAPI patterns)
- The same `AGENTS.md` pointing to the standards

The difference is:

- **A:** "You can read these standards if you want." → A didn't want to, so A used implicit defaults (200 + body for DELETE).
- **B:** "You must enumerate every status code." → Forced question → Looked up std-009 → Found 204 convention.

Forced enumeration → forced consultation → better outcomes.

This is the core finding: **It's not about giving the agent better information. It's about compelling it to use information it already has.**

## Potential criticism #6: "But maybe A's agent is just lazier or less capable."

**Response:**

Both agents are Claude Opus 4.7, same settings, same model. There's no reason to believe one instance of the model is "lazier" than another.

The difference is not capability; it's **incentive structure.** B's agent was given a checklist and told to mark items complete. That creates an incentive to enumerate. A's agent was not.

This is not a criticism of A's agent. It's an observation about what happens when you don't compel enumeration: reasonable defaults are used instead of deliberate choices.

## Potential criticism #7: "But a human developer wouldn't need schemas to do the right thing."

**Response:**

True, a senior human developer wouldn't skip 404 documentation or use 200 for DELETE. But:

1. Not all human developers are senior, and schemas help everyone.
2. Autonomous agents operate at scale (thousands of features). You can't have a senior human review every decision. Schemas automate the "junior developer reviewer" role.
3. The experiment is about **autonomous agent governance**, not about whether humans need guidance.

## What variables we DID control

✓ **Model:** Claude Opus 4.7, high reasoning, 200K context  
✓ **Feature:** Identical spec (soft-delete with 404 handling)  
✓ **Codebase baseline:** Separate branches, but from same ancestor, differing only in schema presence  
✓ **Environment:** Shared PostgreSQL, same ports  
✓ **Session:** Brand new for each track, no history carry-over  
✓ **Fairness checkpoint:** Both agents saw the same proposal text  

## What variables we DID NOT control

✗ **Prompt difference:** Intentional; we wanted to test the effect of forcing schema use.  
✗ **Approval count:** Different by design (schema checklist asks more questions).  
✗ **Agent internal state:** Each chat session is isolated; we can't see the agent's reasoning, only outputs.  

The intentional prompt difference is the point of the experiment. Everything else was controlled.

## How to think about fairness

**Fair != identical.**

Fair means: "We isolated the variable we care about (forced enumeration via schemas) and controlled everything else."

If we had given both agents identical prompts, neither would have used schemas, and there would be no experiment to run.

The fairness is in the **isolation**, not in the prompts being identical.

---

## Further reading

- [FINDINGS.md](../FINDINGS.md) — Three divergences and why they happened
- [PROCESS.md](../PROCESS.md) — Detailed methodology
- [evidence/B/compliance-log.yaml](../evidence/B/compliance-log.yaml) — B's audit trail (A has no equivalent)
