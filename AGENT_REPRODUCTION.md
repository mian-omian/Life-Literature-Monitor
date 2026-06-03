# Agent Reproduction Guide

This guide has two layers:

1. **Operate the existing repository** so a beginner can validate and run the
   current LifeLit project with agent assistance.
2. **Rebuild the current feature set** in ordered implementation phases.

This is supporting guidance. Before implementing, an agent must know which
repository it is in.

- In the **full LifeLit implementation repository**, read the authoritative
  governance docs and current source before editing.
- In the **standalone portal repository**, read the local portal files for
  orientation, then obtain the full implementation repository before making
  implementation claims or code changes.

## Layer 1: Operate the Existing Repository

### Step 1: Identify the Repository Context

If the repository contains `src/lifelit/`, `config/`, `tests/`, `AGENTS.md`,
and `CURRENT_TASK.md`, it is the full implementation repository.

If the repository contains only portal files such as `README.md`,
`ARCHITECTURE.md`, `WORKFLOW.md`, `CONFIG_REFERENCE.md`, `USER_GUIDE.md`,
`AGENT_REPRODUCTION.md`, and `diagrams/`, it is the standalone portal
repository.

### Step 2: Read the Relevant Authority Set

In the full implementation repository, read:


```text
AGENTS.md
CURRENT_TASK.md
TASKS.md
DESIGN.md
ACCEPTANCE_CRITERIA.md
README.md
docs/user_guide.en.md
docs/documentation_inventory.md
```

In the standalone portal repository, read:

```text
README.md
USER_GUIDE.md
ARCHITECTURE.md
WORKFLOW.md
CONFIG_REFERENCE.md
AGENT_REPRODUCTION.md
diagrams/pipeline-flow.md
diagrams/data-model.md
```

Do not treat the standalone portal files as proof of current runtime behavior
until they are checked against the full implementation repository.

### Step 3: Classify the Worktree

Run:

```powershell
git status --short --branch --untracked-files=all
```

Classify pending changes as implementation diffs, documentation diffs,
generated run artifacts, worker artifacts, or user-owned config changes.

In the standalone portal repository, this classification is documentation-only
unless the owner has added implementation files.

### Step 4: Use Safe Inspection Commands

```powershell
uv run lifelit --help
uv run lifelit validate-config --config config
uv run lifelit retrieval simulate --config config
uv run lifelit readiness --config config
uv run lifelit review --dry-run
uv run lifelit notify --dry-run
```

These commands require the full implementation repository and installed
dependencies. They are not available in the standalone portal repository by
itself.

### Step 5: Avoid Unsafe Defaults

Do not run real API, LLM, SMTP, commit, push, deletion, dependency update, or
workflow-changing commands unless the user explicitly authorizes them and the
active task gate allows them.

## Layer 2: Rebuild the Current Feature Set

The rebuild path below is decision-complete enough for an agent to implement in
phases. Each phase should include tests before moving on.

### Phase 0: Project Scaffold

Build:

- Python 3.12 package with `uv`;
- Click CLI entry point `lifelit`;
- `version`, `validate-config`, and placeholder command group structure;
- Ruff, mypy strict mode, and pytest with repository-local basetemp.

Acceptance:

```powershell
uv sync --frozen
uv run lifelit --help
uv run lifelit version
```

### Phase 1: Strict Config

Build Pydantic v2 models for 13 YAML files:

```text
authors.yml
filters.yml
journals.yml
llm.yml
output.yml
privacy.yml
runtime.yml
scoring.yml
seeds.yml
semantic_scholar.yml
sources.yml
storage.yml
topics.yml
```

Required behavior:

- strict validation;
- secret scanning for config values;
- registered scoring signal validation;
- supported section override validation;
- stable identifier validation where required.

Acceptance:

```powershell
uv run lifelit validate-config --config config
uv run pytest tests/test_config.py -q
```

### Phase 2: Domain Models

Build Pydantic models for:

- `RawRecord`
- `CandidatePaper`
- `CanonicalPaper`
- `FilterDecision`
- `SectionScoreResult`
- `LlmTriageResult`
- feedback and run metadata models

Required behavior:

- every `RawRecord` preserves source, entry type, retrieval mode, query/profile
  or author IDs where applicable, retrieved time, source record ID, source URL,
  and raw payload;
- candidate/canonical papers require stable identifiers or title plus source
  evidence;
- section scores are section-local.

Acceptance:

```powershell
uv run pytest tests/test_models.py -q
```

### Phase 3: Retrieval Strategy and Sources

Build:

- offline strategy compiler;
- retrieval simulator;
- PubMed adapter;
- Europe PMC adapter;
- bioRxiv adapter;
- Semantic Scholar Recommendations adapter and date-window post-filter;
- stable-ID tracked-author adapter;
- fixed source-policy/rate-limit/retry layer.

Non-scope:

- Semantic Scholar daily Search unless explicitly activated as a future task;
- runtime pure-name author tracking;
- real API calls in unit tests.

Acceptance:

```powershell
uv run lifelit retrieval simulate --config config
uv run pytest tests/test_retrieval_strategy.py tests/test_pubmed.py tests/test_europe_pmc.py tests/test_biorxiv.py tests/test_semantic_scholar.py tests/test_author_tracked.py -q
```

### Phase 4: Processing

Build:

- normalization from `RawRecord` to `CandidatePaper`;
- identifier-first dedup into `CanonicalPaper`;
- enrichment across S2, OpenAlex, Crossref, Europe PMC, and bioRxiv status;
- shared journal identity matching;
- filtering with technical/bypass/persisted/formal whitelist/configured
  rule/publication/recommendation/preprint/metadata/quality/default stages.

Acceptance:

```powershell
uv run pytest tests/test_normalization.py tests/test_dedup.py tests/test_enrichment.py tests/test_filtering.py -q
```

### Phase 5: Scoring and Clinical Annotation

Build:

- 10 registered signals;
- four macro sections;
- per-topic saturation scoring;
- journal reputation scoring;
- recommendation profile/rank signals;
- section overrides merged at runtime;
- clinical-trial venue annotation.

Acceptance:

```powershell
uv run pytest tests/test_scoring.py tests/test_clinical.py -q
```

### Phase 6: LLM Triage

Build:

- OpenAI-compatible client;
- strict JSON/schema output contract;
- budget manager;
- cache;
- fallback TLDR;
- per-paper recoverable failure handling;
- post-LLM rerank/reband.

Non-scope:

- LLM retrieval;
- automatic config mutation;
- committing raw prompts by default.

Acceptance:

```powershell
uv run pytest tests/test_llm_client.py tests/test_budget.py tests/test_cache.py tests/test_prompts.py tests/test_llm_triage.py tests/test_llm_triage_phase.py -q
```

### Phase 7: Render, State, Review, and Notify

Build:

- `review_data_full.json`;
- curated `review_data.json`;
- Markdown report;
- papers CSV;
- Zotero CSV;
- suppressed CSV;
- run summary;
- Parquet state helpers;
- S2 suppression state;
- Streamlit review UI;
- feedback events/proposals/config drafts;
- SMTP notification dry-run and send paths.

`static_html` remains planned unless a future task implements it.

Acceptance:

```powershell
uv run pytest tests/test_render.py tests/test_state.py tests/test_review_ui.py tests/test_feedback.py tests/test_proposals.py tests/test_notify.py -q
```

### Phase 8: CLI Orchestration, Readiness, and Workflows

Build:

- full `lifelit run`;
- `readiness`;
- `retrieval simulate`;
- feedback commands;
- profile compiler commands;
- config promotion;
- manual import;
- GitHub Validate and Daily Pipeline workflows;
- local validation hook scripts.

Acceptance:

```powershell
uv sync --frozen
uv run lifelit validate-config --config config
uv run lifelit retrieval simulate --config config
uv run lifelit readiness --config config
uv run ruff check .
uv run mypy src
uv run pytest
```

## Reproduction Rules

- Implement one phase at a time.
- Keep examples domain-neutral.
- Use mocks for unit tests.
- Do not commit secrets.
- Do not silently mutate config.
- Do not mark a phase complete until validation passes or the blocker is
  recorded exactly.
