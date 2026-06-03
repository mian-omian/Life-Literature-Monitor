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

## Authority

This portal is supporting documentation. If it conflicts with repository
authority, the repository authority wins in this order:

1. `AGENTS.md`
2. `CURRENT_TASK.md`
3. `TASKS.md`
4. `DESIGN.md`
5. `ACCEPTANCE_CRITERIA.md`
6. `README.md` and `docs/user_guide.en.md`
7. `docs/documentation_inventory.md`
8. this `project-portal/`

Always check the current source under `src/lifelit/` before implementing from
this portal.

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

Read these files in order:

1. `../README.md` — current repository overview and operating model.
2. `../docs/user_guide.en.md` — authoritative user guide.
3. `../docs/getting_started.md` — first-run walkthrough.
4. `../docs/cli_reference.md` — command reference.
5. `ARCHITECTURE.md` — portal architecture map.
6. `WORKFLOW.md` — run lifecycle.
7. `CONFIG_REFERENCE.md` — config field status.
8. `AGENT_REPRODUCTION.md` — how to ask an agent to reproduce or rebuild.

## Ask an Agent Safely

When asking a coding agent to help:

- Tell it to read `AGENTS.md`, `CURRENT_TASK.md`, `TASKS.md`, `DESIGN.md`,
  `ACCEPTANCE_CRITERIA.md`, `README.md`, and `docs/user_guide.en.md` before
  making changes.
- Do not paste API keys, SMTP passwords, tokens, `.env` values, private
  credentials, or raw secret values.
- Ask for read-only inspection first when you are unsure.
- Require a bounded task gate before edits.
- Require validation evidence before review.

## Portal Files

| File | Purpose |
|---|---|
| `ARCHITECTURE.md` | Current architecture and module map |
| `WORKFLOW.md` | Runtime pipeline walkthrough and command risk boundaries |
| `CONFIG_REFERENCE.md` | Source-current config implementation status |
| `USER_GUIDE.md` | Beginner-oriented portal guide |
| `AGENT_REPRODUCTION.md` | Two-layer reproduction guide for agents |
| `diagrams/pipeline-flow.md` | Mermaid pipeline diagram |
| `diagrams/data-model.md` | Mermaid data model diagram |
