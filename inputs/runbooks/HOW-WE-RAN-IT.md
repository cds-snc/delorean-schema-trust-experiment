# How we ran the Schema Trust A/B

A one-page handout so anyone reading the comparison can trust the result.

## The question

If we ask an AI to build the same feature twice — once with our schema-trust
standard, once without — does it actually matter?

## The experiment

**One feature, two agents, one variable changed.**

| | Track A (unguided) | Track B (schema-guided) |
|---|---|---|
| Proposal given to the agent | The same feature spec | The same feature spec **plus one paragraph** requiring schema-trust |
| `delorean/schemas/` available? | No | Yes |
| `AGENTS.md` schema contracts section? | No | Yes |
| Everything else | identical | identical |

The "one paragraph" is the only intentional difference between the two
prompts. Everything an agent sees in its worktree was scoped to match.

## How we kept it fair

- **Same starting point.** Each track forked from its own baseline commit
  containing only the materials that track is allowed to see. No
  cross-contamination.
- **Same model, same settings.** Claude Opus 4.7, high reasoning, 200K
  context, agent mode, brand-new chat session on each side.
- **Same environment.** Both runs hit the same PostgreSQL instance on the
  same ports, sequentially, in two isolated VS Code windows.
- **Full capture.** Every chat turn, every tool call, every file change
  was recorded to disk. The raw artifacts are in `A/` and `B/` next to
  this document.

## What we measured

- **Throughput.** Turns, tool calls, wall-clock time, model credits.
- **Output.** Files changed, lines added/deleted, tests passing, coverage.
- **Behavioural artifacts.** What the produced code actually does when
  exercised in a browser, and what its OpenAPI spec documents.

## What we did not measure

- **Subjective code quality.** We did not ask reviewers to rate either
  output. Both features work; the comparison is about *what* they
  built differently, not which is "prettier."
- **Long-term maintainability.** A single feature in a single afternoon
  cannot speak to drift over months.
- **Generalisation.** This is one feature, one model, one moment. Treat
  the result as a signal, not a proof.

## What's in the archive

- `COMPARISON.md` — the findings.
- `A/` and `B/` — raw evidence per track: summary, diff stats, pytest
  output, git status, manual metrics captured from the chat UI.
- B also contains `compliance-log.yaml` — the schema-audit trail B
  produced for itself. A has no equivalent.
