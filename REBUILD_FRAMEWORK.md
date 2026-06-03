# Rebuild Framework

This file defines how a coding agent should rebuild LifeLit from the standalone
portal repository without drifting from the intended system.

## Goal

Rebuild a local-first LifeLit implementation that matches the portal contracts:

- strict YAML config;
- single-process Click CLI;
- source retrieval adapters with mocked default tests;
- normalization, deduplication, enrichment, filtering, scoring, LLM triage,
  rendering, state, review, feedback, readiness, and notification layers;
- private single-owner operating model;
- no secret values or user-specific research interests in public examples.

The original LifeLit implementation repository is optional audit evidence. It
is not required to start or complete a rebuild from this portal.

## Required Agent Loop

For every milestone:

1. Read `MILESTONES.md` and the relevant contract files.
2. Restate the milestone goal, allowed files, non-scope, and acceptance
   commands in the target implementation workspace.
3. Implement only that milestone.
4. Add or update tests before claiming completion.
5. Run the milestone acceptance commands.
6. Record exact results and unresolved gaps.
7. Stop. Do not begin the next milestone until the user approves.

## Drift Controls

The agent must not:

- invent new product features beyond the contracts;
- silently simplify data models;
- replace strict config with natural-language runtime inference;
- use real API calls in unit tests;
- store secrets in files;
- implement runtime pure-name author tracking;
- make static HTML or Semantic Scholar daily Search active before the explicit
  milestone that implements them;
- compare scores across macro sections;
- let LLM triage retrieve, mutate config, or override hard suppressions.

## Contract File Order

Read these before implementation:

1. `README.md`
2. `MILESTONES.md`
3. `IMPLEMENTATION_SPEC.md`
4. `CONFIG_SCHEMA_CONTRACT.md`
5. `CLI_CONTRACT.md`
6. `DATA_CONTRACTS.md`
7. `WORKFLOW_CONTRACT.md`
8. `TEST_REBUILD_PLAN.md`
9. `AGENT_REPRODUCTION.md`

## Completion Definition

A rebuilt implementation is complete only when:

- every milestone in `MILESTONES.md` is complete;
- every contract file has corresponding tests;
- full validation passes:

```powershell
uv sync --frozen
uv run lifelit validate-config --config config
uv run lifelit retrieval simulate --config config
uv run lifelit readiness --config config
uv run ruff check .
uv run mypy src
uv run pytest
```

- no real API, LLM, SMTP, or secret-dependent test is required for default
  validation.
