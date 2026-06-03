# Implementation Spec

This file describes the required subsystem architecture for a LifeLit rebuild.

## Runtime Shape

LifeLit is a single-process Python CLI application. The primary entry point is:

```powershell
uv run lifelit run --config config
```

The run path is:

```text
load strict config
-> compute runtime window
-> compile retrieval strategy
-> retrieve enabled sources unless dry_run
-> normalize
-> deduplicate
-> enrich
-> filter
-> score by section
-> annotate clinical-trial venue signals
-> run bounded LLM triage
-> rerank/reband if LLM adjusted scores
-> render outputs
-> update state
-> run health gates
-> promote healthy snapshot
-> notify when configured
```

## Required Modules

| Module | Required responsibility |
|---|---|
| `config.py` | Strict YAML config models and loader |
| `models.py` | Domain/data models and validation |
| `retrieval_strategy.py` | Offline strategy compiler and simulator structures |
| `pubmed.py` | PubMed adapter, mocked by default tests |
| `europe_pmc.py` | Europe PMC adapter and partial-failure handling |
| `biorxiv.py` | bioRxiv adapter and local topic filtering |
| `semantic_scholar.py` | Recommendations retrieval, enrichment helpers, date filtering |
| `author_tracked.py` | Stable-ID tracked-author retrieval |
| `normalization.py` | RawRecord to CandidatePaper conversion |
| `dedup.py` | CandidatePaper to CanonicalPaper merge |
| `enrichment.py` | S2/OpenAlex/Crossref/Europe PMC/bioRxiv enrichment |
| `journal_matching.py` | Shared journal identity matching |
| `filtering.py` | Guardrails plus configured filter rules |
| `scoring.py` | 10-signal section-first scoring |
| `clinical.py` | Clinical-trial venue annotation |
| `llm_client.py` | OpenAI-compatible client |
| `prompts.py` | Prompt and JSON contract |
| `budget.py` | Token/cost budget accounting |
| `cache.py` | LLM cache |
| `fallback_tldr.py` | Deterministic fallback TLDR |
| `llm_triage.py` | Bounded section-aware triage |
| `render.py` | Review/report/CSV/Zotero/suppressed/summary outputs |
| `state/` | Parquet/YAML state helpers |
| `feedback.py` | Feedback event handling |
| `proposals.py` | Proposal and config-draft handling |
| `profile_agent.py` | Deterministic profile text parser |
| `profile_resolver.py` | Identifier resolver abstraction |
| `profile_compiler.py` | Profile to config-draft compiler |
| `review_ui.py` | Streamlit review UI |
| `notify.py` | Notification preview and SMTP send |
| `readiness.py` | Offline readiness and effective-config summary |
| `run_health.py` | Health and promotion gates |
| `source_reliability.py` | Structured source failure diagnostics |

## Non-Negotiable Semantics

- Runtime config is strict YAML, not natural language.
- Config-changing feedback creates proposals or config drafts, not direct
  mutation.
- `RawRecord` preserves source provenance.
- Dedup is identifier-first.
- Filtering is not scoring.
- Scoring is section-local.
- LLM triage is bounded and cannot retrieve, mutate config, or override hard
  suppressions.
- Default tests use mocks/fixtures, not real external services.
- Static HTML and Semantic Scholar daily Search remain deferred unless a later
  milestone explicitly implements them.

## Deferred Surfaces

Keep these visible as planned/deferred unless explicitly implemented:

- static HTML output;
- Semantic Scholar daily Search;
- complete boundary-seed calibration semantics;
- public-release privacy hardening beyond private single-owner assumptions.
