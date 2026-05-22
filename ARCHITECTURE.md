# Architecture

## Design Overview

LifeLit is a **single-process, batch-oriented pipeline** orchestrated by a
Click CLI. Each invocation of `lifelit run` executes the full pipeline top to
bottom: retrieve → normalize → deduplicate → enrich → filter → score → triage →
render → persist. There is no long-running server, no task queue, and no
database beyond Parquet files on disk.

The pipeline is designed to run inside a **GitHub Actions workflow** on a daily
cron schedule, but can also run locally during development and configuration
iteration.

---

## Module Map

```
src/lifelit/
├── cli.py                  # Click CLI: 19 commands, pipeline orchestrator
├── config.py               # Pydantic v2 models for all 13 config files
├── models.py               # Core domain models (RawRecord, CanonicalPaper, etc.)
│
├── retrieval_strategy.py   # Compiles config into retrieval plans per source
├── pubmed.py               # PubMed E-Utilities API client
├── europe_pmc.py           # Europe PMC REST API client
├── biorxiv.py              # bioRxiv API client
├── semantic_scholar.py     # Semantic Scholar Recommendations + Search API
├── author_tracked.py       # ORCID-based author paper retrieval
├── retrieval.py            # Raw record parsing / manual import
│
├── normalization.py        # Journal name normalization + legacy config migration
├── dedup.py                # Cross-source deduplication by DOI/PMID/title
├── enrichment.py           # Metadata enrichment (S2, OpenAlex, Crossref)
├── filtering.py            # 9-rule hardcoded guardrail pipeline
├── scoring.py              # 11-signal multi-factor scoring engine
│
├── llm_client.py           # OpenAI-compatible LLM client abstraction
├── llm_triage.py           # LLM-based paper triage runner
├── prompts.py              # LLM prompt templates
├── cache.py                # Parquet-backed LLM response cache
├── budget.py               # Token + cost budget manager
├── fallback_tldr.py        # Rule-based TLDR extraction fallback
│
├── render.py               # Output renderers (JSON, MD, CSV, Zotero, summary)
├── review_ui.py            # Streamlit local review UI
├── notify.py               # SMTP email notification
│
├── state/
│   ├── __init__.py         # Parquet state read/write (seen_papers, paper_status, etc.)
│   └── suppression.py      # S2 recommendation seen/rejected suppression
│
├── run_health.py           # 4 health gates for output promotion
├── clinical.py             # Clinical trial observation annotation
├── source_reliability.py   # Source failure tracking and diagnostics
├── readiness.py            # Offline readiness / smoke checks for CI
│
├── profile_agent.py        # NL profile text parser
├── profile_resolver.py     # Identifier resolution (DOI, PMID, ORCID, etc.)
├── profile_compiler.py     # Profile → config draft compiler
│
├── feedback.py             # Feedback event load / apply / undo
└── proposals.py            # Config proposal generation and config draft management
```

---

## Key Design Decisions

### 1. Config as Source of Truth

All pipeline behavior is declared in 13 YAML files under `config/`. Each file
maps to a strict Pydantic v2 model in `config.py`. The `validate-config`
command checks cross-file integrity (e.g., scoring signal names referenced in
section overrides must exist in the registered signals list).

The config surface is intentionally broad — users should never need to edit
Python to change what papers they see.

### 2. Snapshot-and-Promote Output Model

```
outputs/
├── latest/           # Symlink/copy of the most recent healthy run
├── 2026-05-22T14-07/ # Timestamped run snapshots
├── 2026-05-21T14-07/
└── ...
```

Each run writes to `outputs/<run_id>/`. After the pipeline completes, health
gates run (`run_health.py`). If all gates pass, the snapshot is "promoted" by
copying its contents to `outputs/latest/`. Old snapshots beyond `retain_runs`
(30) are pruned.

This means `outputs/latest/` is always a valid, healthy run — safe to commit
and safe for downstream consumers to read.

### 3. Parquet State, Not a Database

Three Parquet files track run-to-run state:

| File | Schema | Purpose |
|---|---|---|
| `state/seen_papers.parquet` | dedup_key, first_seen_run, last_seen_run, source_ids | Tracks every paper ever seen |
| `state/paper_status.parquet` | dedup_key, status, suppressed_reason, updated_at | Per-paper curation decisions |
| `state/run_index.parquet` | run_id, timestamp, mode, paper_count, health | Historical run log |
| `state/llm_cache.parquet` | prompt_hash, response, model, created_at | LLM response cache |

Parquet was chosen because: (a) it's a file, not a process — perfect for CI
runners with no persistent storage; (b) it's columnar, so reading subsets is
fast; (c) PyArrow handles it natively with zero configuration.

### 4. Multi-Source Retrieval with Role-Based Strategy

Rather than a single unified search, each source contributes papers through a
**compiled retrieval strategy** (`retrieval_strategy.py`):

- **PubMed** → formal literature search with MeSH terms and date windows
- **Europe PMC** → supplementary search covering open-access content
- **bioRxiv** → preprint retrieval with topic-term filtering
- **Semantic Scholar** → recommendation engine driven by seed papers (DOI → similar papers)
- **Author Tracked** → papers by specific ORCID-tracked researchers

Each source's compiled strategy carries its query clauses, date ranges, and
topic terms. The CLI iterates over enabled sources, calls each retriever, and
aggregates raw records.

### 5. Hardcoded Filter Rules by Design

The filtering pipeline (`filtering.py`) uses 9 hardcoded rules rather than a
config-driven rule engine. This was an explicit choice: the rules encode
domain-specific guardrails (e.g., "preprints without abstracts and without
venue names are suppressed") that are stable and well-understood. A
config-driven `FilterRule` model exists in the schema for future extension but
is not wired in.

### 6. LLM Triage with Budget Guardrails

LLM triage is optional and heavily guarded:

- **Eligible sections** are config-controlled (`llm.yml: eligible_sections`)
- **Per-paper token limit** (`budget.per_paper_token_limit`) truncates prompts
- **Per-run token/cost caps** (`budget.per_run_token_limit`,
  `budget.per_run_cost_limit`) halt triage when exceeded
- **Score adjustment clamp** (`max_score_adjustment`) bounds LLM influence on
  final ranking
- **Response cache** (`state/llm_cache.parquet`) avoids re-triaging the same
  paper across runs
- **Fallback TLDR** (`fallback_tldr.py`) provides rule-based one-line summaries
  when LLM is unavailable or budget is exhausted

### 7. Scoring as Configurable Signal Aggregation

The scoring engine computes 11 independent signals, each returning a value in
[0, 1], then aggregates them with per-section weights:

```
final_score = Σ (signal_value × signal_weight) / Σ signal_weights
```

Signals are grouped into **sections** (published, preprinted,
author_tracked_manual, semantic_recommendations), each with its own active
signal set and default weights. The signal set is hardcoded in
`REGISTERED_SIGNALS`; section weights use hardcoded `PRESET_CONTEXTS` (the
config-driven `section_overrides` schema exists but is not yet wired).

---

## Dependency Graph

```
config.py (Pydantic models)
  ├── consumed by: cli.py, retrieval_strategy.py, scoring.py, enrichment.py
  └── validated by: validate-config CLI command

models.py (domain models)
  └── consumed by: every pipeline module

retrieval_strategy.py
  ├── consumes: config.topics, config.journals, config.seeds, config.sources
  └── consumed by: cli.py (orchestrator)

scoring.py
  ├── consumes: config.scoring, config.journals, config.topics
  └── consumed by: cli.py (orchestrator)

llm_triage.py
  ├── consumes: config.llm, llm_client, budget, cache, prompts
  └── consumed by: cli.py (orchestrator)

state/__init__.py
  ├── consumes: config.storage
  └── consumed by: cli.py (both during run and for review UI)

render.py
  ├── consumes: config.output
  └── consumed by: cli.py (render phase)
```

The `cli.py` module is the sole orchestrator. No pipeline module imports
another pipeline module directly — all coupling flows through `cli.py`, which
sequences the phases and passes data between them.
