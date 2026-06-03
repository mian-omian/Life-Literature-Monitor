# Rebuild Milestones

Milestones are sequential. Do not skip ahead. Do not merge milestones unless
the user explicitly authorizes a larger implementation batch.

## M0 — Scaffold and Tooling

Goal: create a Python 3.12 package with `uv`, Click, Ruff, mypy, pytest, and a
minimal `lifelit` CLI.

Required outputs:

- `pyproject.toml`
- `src/lifelit/__init__.py`
- `src/lifelit/__main__.py`
- `src/lifelit/cli.py`
- `tests/`

Acceptance:

```powershell
uv sync --frozen
uv run lifelit --help
uv run lifelit version
uv run ruff check .
uv run mypy src
uv run pytest
```

## M1 — Strict Config System

Goal: implement strict Pydantic v2 loading for 13 YAML files.

Contract: `CONFIG_SCHEMA_CONTRACT.md`.

Acceptance:

```powershell
uv run lifelit validate-config --config config
uv run pytest tests/test_config.py -q
```

## M2 — Domain Models and Data Contracts

Goal: implement core Pydantic models and validation for records, papers,
scores, triage results, feedback, state, and review data.

Contract: `DATA_CONTRACTS.md`.

Acceptance:

```powershell
uv run pytest tests/test_models.py tests/test_fixtures.py -q
```

## M3 — Retrieval Strategy and Source Adapters

Goal: implement offline retrieval strategy compilation and mocked retrievers
for PubMed, Europe PMC, bioRxiv, Semantic Scholar Recommendations, and tracked
authors.

Non-scope:

- Semantic Scholar daily Search;
- real API calls in unit tests;
- pure-name author tracking.

Acceptance:

```powershell
uv run lifelit retrieval simulate --config config
uv run pytest tests/test_retrieval_strategy.py tests/test_pubmed.py tests/test_europe_pmc.py tests/test_biorxiv.py tests/test_semantic_scholar.py tests/test_author_tracked.py -q
```

## M4 — Processing Pipeline

Goal: implement normalization, deduplication, enrichment, journal identity
matching, filtering, and source reliability diagnostics.

Acceptance:

```powershell
uv run pytest tests/test_normalization.py tests/test_dedup.py tests/test_enrichment.py tests/test_filtering.py tests/test_source_reliability.py -q
```

## M5 — Section-First Scoring

Goal: implement four macro sections, 10 registered scoring signals, section
contexts, section overrides, ranking, banding, and clinical-trial venue
annotation.

Acceptance:

```powershell
uv run pytest tests/test_scoring.py tests/test_clinical.py -q
```

## M6 — Bounded LLM Triage

Goal: implement OpenAI-compatible LLM client, prompt contract, JSON schema
validation, budget, cache, fallback TLDR, recoverable per-paper failures, and
post-LLM rerank/reband.

Acceptance:

```powershell
uv run pytest tests/test_llm_client.py tests/test_budget.py tests/test_cache.py tests/test_prompts.py tests/test_fallback_tldr.py tests/test_llm_triage.py tests/test_llm_triage_phase.py -q
```

## M7 — Rendering, State, Feedback, Review, and Notify

Goal: implement review data, Markdown/CSV/Zotero/suppressed/run-summary
outputs, Parquet/YAML state, feedback events, proposals/config drafts, review
UI dry-run/launch behavior, and notification dry-run/send behavior.

Acceptance:

```powershell
uv run pytest tests/test_render.py tests/test_state.py tests/test_feedback.py tests/test_proposals.py tests/test_review_ui.py tests/test_notify.py -q
```

## M8 — CLI Orchestration and Readiness

Goal: wire all layers through the CLI, implement readiness and retrieval
simulation summaries, enforce run/dry-run behavior, health gates, and local
validation scripts.

Contracts: `CLI_CONTRACT.md`, `WORKFLOW_CONTRACT.md`.

Acceptance:

```powershell
uv run lifelit --help
uv run lifelit validate-config --config config
uv run lifelit retrieval simulate --config config
uv run lifelit readiness --config config
uv run lifelit review --dry-run
uv run lifelit notify --dry-run
uv run pytest tests/test_cli.py tests/test_readiness.py tests/test_run_health.py -q
```

## M9 — Full Validation Closure

Goal: prove the rebuilt repository is complete for the contracted feature set.

Acceptance:

```powershell
uv sync --frozen
uv run lifelit validate-config --config config
uv run lifelit retrieval simulate --config config
uv run lifelit readiness --config config
uv run ruff check .
uv run mypy src
uv run pytest
git diff --check
```

Completion report must list implemented files, tests, validation results,
known limitations, and deferred surfaces.
