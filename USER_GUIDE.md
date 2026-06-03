# Beginner User Guide

This guide is the portal's beginner path. In the full LifeLit implementation
repository, the authoritative user manual is `../docs/user_guide.en.md`. In the
standalone portal repository, that file is not local; use this guide and the
other portal files as the contract for agent-led reconstruction.

## 1. Install and Inspect

Install dependencies:

```powershell
uv sync --frozen
```

Check the CLI:

```powershell
uv run lifelit --help
uv run lifelit version
```

## 2. Validate Configuration

```powershell
uv run lifelit validate-config --config config
```

This checks the 13 YAML files under `config/`. Passing validation means the
schema is accepted; it does not mean every field is safe to change casually.

## 3. Understand Before Running

Use offline inspection first:

```powershell
uv run lifelit retrieval simulate --config config
uv run lifelit readiness --config config
uv run lifelit review --dry-run
uv run lifelit notify --dry-run
```

Readiness may return `warn`. A warning is not automatically a failure; inspect
the reported source, output, privacy, and effective-config details.

## 4. Know What Writes Files

These commands can write local operating artifacts:

```powershell
uv run lifelit run --config config --mode dry_run
uv run lifelit run --config config
uv run lifelit feedback apply
uv run lifelit feedback config-draft
uv run lifelit profile compile
uv run lifelit config-promote
```

Real runs may also call literature APIs, an LLM provider, and SMTP depending on
enabled config and environment variables.

## 5. Review Outputs

After a healthy run, inspect:

```text
outputs/latest/report.md
outputs/latest/review_data.json
outputs/latest/run_summary.json
outputs/latest/papers.csv
outputs/latest/zotero.csv
outputs/latest/suppressed.csv
```

`review_data_full.json` is an audit artifact for the broader keep set before
primary reading curation.

## 6. Open the Review UI

Dry-run first:

```powershell
uv run lifelit review --dry-run
```

Launch when ready:

```powershell
uv run lifelit review
```

The review UI writes feedback events under `feedback/events/`. It does not
silently mutate live config.

## 7. Change Configuration Safely

Preferred path:

```powershell
uv run lifelit profile compile --input-file profile/natural_language.md --output-dir state/config_draft
uv run lifelit profile validate --draft-path state/config_draft/config_draft.yml
uv run lifelit profile diff --draft-path state/config_draft/config_draft.yml --config-dir config
uv run lifelit config-promote --dry-run
uv run lifelit config-promote
uv run lifelit validate-config --config config
```

Feedback path:

```powershell
uv run lifelit feedback apply
uv run lifelit feedback config-draft
uv run lifelit profile validate --draft-path state/config_draft/config_draft.yml
uv run lifelit profile diff --draft-path state/config_draft/config_draft.yml --config-dir config
```

Do not put API keys, SMTP passwords, tokens, or `.env` values in config files.

## 8. Ask an Agent for Help

Use a prompt like:

```text
If this is the full LifeLit implementation repository, read AGENTS.md,
CURRENT_TASK.md, TASKS.md, DESIGN.md, ACCEPTANCE_CRITERIA.md, README.md,
docs/user_guide.en.md, and docs/documentation_inventory.md before edits.

If this is the standalone Life-Literature-Monitor portal repository, read the
local README.md, USER_GUIDE.md, ARCHITECTURE.md, WORKFLOW.md,
CONFIG_REFERENCE.md, REBUILD_FRAMEWORK.md, MILESTONES.md,
IMPLEMENTATION_SPEC.md, CONFIG_SCHEMA_CONTRACT.md, CLI_CONTRACT.md,
DATA_CONTRACTS.md, WORKFLOW_CONTRACT.md, TEST_REBUILD_PLAN.md,
AGENT_REPRODUCTION.md, and diagrams first. Create a new implementation
workspace and rebuild from the portal milestones. Treat the original LifeLit
implementation repository, if available, as optional audit evidence.

Start with read-only review. If you are rebuilding from the standalone portal,
create your own task checklist before edits and validate each phase before
moving on.
```

For development work in the original implementation repository, the agent must
obey the task gate. For standalone-portal reconstruction, `MILESTONES.md` is
the task gate: implement one milestone at a time, run its acceptance commands,
and keep examples domain-neutral.
