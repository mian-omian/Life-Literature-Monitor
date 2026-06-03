# LifeLit Project Portal

LifeLit is a private, local-first literature monitor for a single owner. It
collects newly indexed or recommended life-science papers, normalizes and
deduplicates them, enriches metadata, filters low-quality records, scores papers
inside section-local reading surfaces, optionally runs bounded LLM triage, and
renders review artifacts.

This portal is written for two audiences:

1. **Beginners** who need a guided path for understanding and running the
   existing repository.
2. **Coding agents** that need a source-current map for reproducing the current
   feature set without copying stale historical specifications.

## Repository Contexts

This content can appear in two different places:

- **Full LifeLit implementation repository**: this portal is located at
  `project-portal/`, next to implementation files such as `AGENTS.md`,
  `CURRENT_TASK.md`, `TASKS.md`, `DESIGN.md`, `ACCEPTANCE_CRITERIA.md`,
  `src/lifelit/`, `docs/`, `config/`, and `tests/`.
- **Standalone portal repository**: this portal is the repository root, such as
  `mian-omian/Life-Literature-Monitor`. In that repository, files like
  `AGENTS.md`, `CURRENT_TASK.md`, `src/lifelit/`, and `docs/user_guide.en.md`
  are upstream implementation-repository files, not local portal files.

When you only have the standalone portal, the portal files are the rebuild
contract. Ask an agent to use them to scaffold a new implementation repository
from scratch. The full LifeLit implementation repository is useful for optional
audit and source comparison, but it is not a prerequisite for rebuilding from
this portal.

## Authority

In the full implementation repository, this portal is supporting documentation.
If it conflicts with implementation-repository authority, the implementation
authority wins in this order:

1. `AGENTS.md`
2. `CURRENT_TASK.md`
3. `TASKS.md`
4. `DESIGN.md`
5. `ACCEPTANCE_CRITERIA.md`
6. `README.md` and `docs/user_guide.en.md`
7. `docs/documentation_inventory.md`
8. this portal

In the standalone portal repository, the local authority order is:

1. `README.md`
2. `REBUILD_FRAMEWORK.md`
3. `MILESTONES.md`
4. `IMPLEMENTATION_SPEC.md`
5. `CONFIG_SCHEMA_CONTRACT.md`
6. `CLI_CONTRACT.md`
7. `DATA_CONTRACTS.md`
8. `WORKFLOW_CONTRACT.md`
9. `TEST_REBUILD_PLAN.md`
10. `AGENT_REPRODUCTION.md`
11. `USER_GUIDE.md`
12. `ARCHITECTURE.md`
13. `WORKFLOW.md`
14. `CONFIG_REFERENCE.md`
15. `diagrams/`

The standalone portal is intended to be enough for beginner orientation and
agent-led reconstruction. If the full LifeLit implementation repository is
available, use `src/lifelit/` and the authoritative docs to audit exact current
behavior; if it is not available, rebuild from this portal's phase
specifications and validation commands.

## What LifeLit Does

At a high level, one run performs this pipeline:

```text
strict config
-> retrieval strategy
-> source retrieval
-> normalization
-> deduplication
-> enrichment
-> filtering
-> section-local scoring
-> bounded LLM triage
-> output rendering
-> state update
-> health gates and notification
```

The runtime is a single-process Python CLI. There is no backend server, task
queue, database service, or SaaS application. State is stored in local files,
primarily Parquet and YAML/JSON artifacts.

## First Safe Commands

These commands are safe orientation checks. They do not perform real literature
retrieval, LLM calls, or SMTP sends:

```powershell
uv run lifelit --help
uv run lifelit validate-config --config config
uv run lifelit retrieval simulate --config config
uv run lifelit readiness --config config
uv run lifelit review --dry-run
uv run lifelit notify --dry-run
```

These commands can write operating artifacts or call external services:

```powershell
uv run lifelit run --config config
uv run lifelit run --config config --mode dry_run
uv run lifelit review
uv run lifelit notify
uv run lifelit config-promote
```

`dry_run` skips retrieval, but the current run path still exercises later
pipeline stages and can write local output/state artifacts. Treat it as a local
pipeline run, not a read-only inspection command.

## Beginner Learning Path

If you are in the full implementation repository, read these files in order:

1. `../README.md` — current repository overview and operating model.
2. `../docs/user_guide.en.md` — authoritative user guide.
3. `../docs/getting_started.md` — first-run walkthrough.
4. `../docs/cli_reference.md` — command reference.
5. `ARCHITECTURE.md` — portal architecture map.
6. `WORKFLOW.md` — run lifecycle.
7. `CONFIG_REFERENCE.md` — config field status.
8. `AGENT_REPRODUCTION.md` — how to ask an agent to reproduce or rebuild.

If you are in the standalone portal repository, read these local files in
order:

1. `README.md`
2. `REBUILD_FRAMEWORK.md`
3. `MILESTONES.md`
4. `IMPLEMENTATION_SPEC.md`
5. `CONFIG_SCHEMA_CONTRACT.md`
6. `CLI_CONTRACT.md`
7. `DATA_CONTRACTS.md`
8. `WORKFLOW_CONTRACT.md`
9. `TEST_REBUILD_PLAN.md`
10. `AGENT_REPRODUCTION.md`
11. `USER_GUIDE.md`
12. `ARCHITECTURE.md`
13. `WORKFLOW.md`
14. `CONFIG_REFERENCE.md`
15. `diagrams/pipeline-flow.md`
16. `diagrams/data-model.md`

## Ask an Agent Safely

When asking a coding agent to help:

- If working in the full implementation repository, tell it to read
  `AGENTS.md`, `CURRENT_TASK.md`, `TASKS.md`, `DESIGN.md`,
  `ACCEPTANCE_CRITERIA.md`, `README.md`, `docs/user_guide.en.md`, and
  `docs/documentation_inventory.md` before making changes.
- If working from the standalone portal repository, tell it to read all local
  portal files first, create a new implementation workspace, and rebuild
  LifeLit milestone by milestone from `REBUILD_FRAMEWORK.md`,
  `MILESTONES.md`, and the contract files. If the original implementation
  repository is available, use it only as optional audit evidence.
- Do not paste API keys, SMTP passwords, tokens, `.env` values, private
  credentials, or raw secret values.
- Ask for read-only inspection first when you are unsure.
- Require a bounded task gate before edits.
- Require validation evidence before review.

## Portal Files

| File | Purpose |
|---|---|
| `REBUILD_FRAMEWORK.md` | Agent rebuild rules, drift controls, completion definition |
| `MILESTONES.md` | Sequential rebuild milestones and acceptance commands |
| `IMPLEMENTATION_SPEC.md` | Required subsystem architecture |
| `CONFIG_SCHEMA_CONTRACT.md` | Config files, schema areas, validation expectations |
| `CLI_CONTRACT.md` | Required commands and command behavior |
| `DATA_CONTRACTS.md` | Core models, outputs, state, provenance |
| `WORKFLOW_CONTRACT.md` | Run, promotion, Actions, and feedback workflow semantics |
| `TEST_REBUILD_PLAN.md` | Required test areas and full validation |
| `ARCHITECTURE.md` | Current architecture and module map |
| `WORKFLOW.md` | Runtime pipeline walkthrough and command risk boundaries |
| `CONFIG_REFERENCE.md` | Source-current config implementation status |
| `USER_GUIDE.md` | Beginner-oriented portal guide |
| `AGENT_REPRODUCTION.md` | Two-layer reproduction guide for agents |
| `diagrams/pipeline-flow.md` | Mermaid pipeline diagram |
| `diagrams/data-model.md` | Mermaid data model diagram |
