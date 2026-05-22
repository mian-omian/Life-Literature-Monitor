# Workflow

## Daily Run Lifecycle

A LifeLit run is a single execution of `lifelit run --config config`. It
follows a strict phase order with no branching or parallelism at the pipeline
level (individual retrievers run sequentially for API rate-limit safety).

### Trigger Sources

| Trigger | Mechanism | Mode |
|---|---|---|
| Scheduled daily run | GitHub Actions cron: `7 6 * * *` UTC (≈14:07 CST) | `daily` |
| Manual workflow dispatch | `workflow_dispatch` with mode selector | `daily`, `dry_run`, `backfill`, `manual` |
| Local development | `uv run lifelit run --config config` | `daily` (default) |
| Simulated retrieval | `uv run lifelit retrieval simulate` | offline / dry |

---

## Phase-by-Phase Walkthrough

### Phase 0: Config Loading & Validation

```
config/ (13 YAML files)
  │
  ├── sources.yml ──────────► SourcesConfig
  ├── runtime.yml ──────────► RuntimeConfig
  ├── output.yml ───────────► OutputConfig
  ├── filters.yml ──────────► FiltersConfig
  ├── privacy.yml ──────────► PrivacyConfig
  ├── llm.yml ──────────────► LlmConfig
  ├── storage.yml ──────────► StorageConfig
  ├── seeds.yml ────────────► SeedsConfig
  ├── scoring.yml ──────────► ScoringConfig
  ├── journals.yml ─────────► JournalsConfig
  ├── authors.yml ──────────► AuthorsConfig
  ├── topics.yml ───────────► TopicsConfig
  └── semantic_scholar.yml ─► SemanticScholarConfig
                                   │
                                   ▼
                              AppConfig
                         (Pydantic v2 validated)
```

Each YAML file is loaded and validated against its Pydantic model. Type errors,
missing required fields, and cross-reference violations are caught here before
any network calls happen. The `validate-config` command runs this phase in
isolation for CI use.

### Phase 1: Readiness Checks

Before making any API calls, the pipeline verifies:

1. Required environment variables are set (`NCBI_API_KEY`,
   `SEMANTIC_SCHOLAR_API_KEY`, etc.) per the enabled sources
2. Network reachability to each API endpoint
3. State files (`seen_papers.parquet`, `paper_status.parquet`,
   `run_index.parquet`) exist or can be initialized
4. Output directory is writable

This is implemented in `readiness.py` and runs as a separate CI step in
`daily.yml` before the pipeline step.

### Phase 2: Retrieval Strategy Compilation

`retrieval_strategy.py` compiles the declarative config into executable
retrieval plans:

```
topics.yml entries
  │
  ├── pubmed_mesh_terms ─────► PubMed [MeSH Terms] clauses
  ├── pubmed_free_text_roots ► PubMed [All Fields] clauses
  ├── europe_pmc_title_abstract_roots ► Europe PMC TITLE/ABSTRACT clauses
  ├── biorxiv_filter_terms ──► bioRxiv post-retrieval filter
  ├── scoring_terms ─────────► Topic signal scoring
  └── llm_context_terms ─────► LLM prompt context injection

seeds.yml positive_seeds ────► Semantic Scholar recommendation input

journals.yml (formal_retrieval role) ► PubMed/EPMC journal filter
```

The compiled strategy is a set of dataclass instances (`PubmedCompiledStrategy`,
`EuropePMCCompiledStrategy`, etc.) carrying the exact query strings and
parameters for each source.

### Phase 3: Retrieval

Each enabled source is called sequentially. Failures are caught per-source and
do not halt the pipeline (unless `partial_failure_policy` is set to `fail`).

```
PubMed ────────── esearch → efetch (batch) ─► RawRecord[]
Europe PMC ────── search → full text ───────► RawRecord[]
bioRxiv ───────── date-range API ───────────► RawRecord[]
Semantic Scholar  recommendations API ──────► RawRecord[]
Author Tracked ── ORCID → S2 author API ────► RawRecord[]
```

Date windows are controlled by `runtime.yml`:
- `newest_offset_days: 1` — exclude papers from today (indexing delay)
- `oldest_offset_days: 7` — look back one week

### Phase 4: Normalization

Raw records from all sources are converted to `CanonicalPaper` instances:

- Journal names are matched against `journals.yml` by name, alias, ISSN, and
  eISSN, then replaced with their canonical form
- Author names are normalized (whitespace, Unicode)
- Date strings are parsed into `date` objects
- Entry types are classified: `author_tracked`, `formal_publication`,
  `preprint`, `semantic_recommendation`, `manual_import`

### Phase 5: Deduplication

Papers are deduplicated across sources using a priority chain:

1. **DOI match** (exact, case-insensitive) — strongest signal
2. **PMID match** — PubMed ID is unique per paper
3. **Title hash match** — normalized title (lowercase, punctuation-stripped)
   as fallback

When duplicates are found, the entry with the richest metadata (most non-empty
fields) is kept, and the others' source IDs are merged into it.

### Phase 6: Enrichment

Each canonical paper is enriched with external metadata:

| Enrichment | Source | Fields Added |
|---|---|---|
| TLDR summaries | Semantic Scholar API | `tldr` |
| Citation count | Semantic Scholar API | `citation_count` |
| Open Access status | OpenAlex API | `is_open_access` |
| Published version DOI | Crossref API | `published_version_doi` (for preprints) |
| Additional metadata | Crossref API | journal, date, author refinements |

Enrichment is opportunistic — failures are logged but do not block the
pipeline.

### Phase 7: Filtering

Nine hardcoded rules run in order. Each rule can keep or suppress a paper.
Once suppressed by any rule, a paper is excluded from scoring and output
(except the suppressed CSV).

| # | Rule | Effect |
|---|---|---|
| 1 | Technical exclude | Drop papers missing title, ID, and authors |
| 2 | Author-tracked bypass | Author-tracked papers skip suppression |
| 3 | Manual-import bypass | Manually imported papers skip suppression |
| 4 | Hard suppress | Re-apply state-persisted suppression |
| 5 | Publication type | Drop editorials, letters, retractions, etc. |
| 6 | Recommendation guardrails | Drop S2 recs with insufficient signal |
| 7 | Preprint guardrails | Drop preprints with no abstract and no venue |
| 8 | Metadata completeness | Drop papers missing both abstract and journal |
| 9 | Technical quality | Multi-dimension quality check |

### Phase 8: Scoring

The 11-signal scoring engine computes per-paper scores within four sections:

| Section | Active Signals | Default Context |
|---|---|---|
| `author_tracked_manual` | entry, topic, recency, metadata_quality, access, preprint_status, state, penalty | Higher base weight on entry_signal |
| `published` | All 11 signals | Higher weight on journal_signal |
| `preprinted` | All 11 signals | Higher weight on journal_signal; preprint_status_signal active |
| `semantic_recommendations` | entry, topic, recommendation_profile, recommendation_rank, recency, metadata_quality, access, state, penalty | Relies on recommendation signals |

Each signal returns a value in `[0, 1]`. The final score is a weighted average
with per-section default weights (configurable via `scoring.yml` section
overrides, though this path is not yet wired to runtime).

Papers are ranked within their section and assigned a **score band**:
- `high` — top tier, eligible for LLM triage
- `medium` — middle tier, included in outputs
- `low` — bottom tier, included but de-emphasized

### Phase 9: Clinical Trial Annotation

Papers are checked against journals marked with `clinical_trial_high_level:
true` in `journals.yml`. Matches are annotated with venue name and match method
(name, alias, ISSN, or eISSN). The annotation is included in rendered outputs.

### Phase 10: LLM Triage (Optional)

If `eligible_sections` is non-empty and an LLM provider is configured:

1. **Selection**: Top papers from eligible sections, up to
   `max_papers_per_section` per section and `max_papers_per_run` total
2. **Budget check**: Token and cost counters from `budget.py` are consulted
   before each API call
3. **Cache lookup**: Prompt hash is checked against `llm_cache.parquet`
4. **LLM call**: The paper's title, abstract, and topic context are sent to the
   LLM with a structured prompt asking for score adjustment and TLDR
5. **Score adjustment**: The LLM's adjustment is clamped to
   `[-max_score_adjustment, +max_score_adjustment]`
6. **Fallback**: If the LLM is unavailable, `fallback_tldr.py` generates a
   rule-based one-line summary from the abstract

### Phase 11: Output Rendering

Six output formats are rendered (each controlled by `output.yml` toggles):

| Output | Format | Contents |
|---|---|---|
| `review_data.json` | JSON | Full structured review data: papers, scores, annotations, triage results, run metadata |
| `report.md` | Markdown | Human-readable daily briefing with sections, score bands, TLDRs |
| `papers.csv` | CSV | Flat table of all papers with scores and metadata |
| `zotero.csv` | CSV | Zotero-importable format with collection assignments |
| `suppressed.csv` | CSV | Papers that were filtered out, with suppression reasons |
| `run_summary.json` | JSON | Run-level statistics: counts, timing, health, diagnostics |

### Phase 12: State Update

Four Parquet files are atomically written:

1. **seen_papers.parquet** — new papers are added; existing papers get
   `last_seen_run` updated
2. **paper_status.parquet** — user curation decisions from the Review UI are
   preserved; new papers get `status: active`
3. **run_index.parquet** — a new row is appended with run metadata
4. **llm_cache.parquet** — new LLM responses are appended (if
   `commit_llm_cache` is enabled in privacy config)

### Phase 13: Health Gates

Four gates must pass for the run to be promoted to `outputs/latest/`:

| Gate | Checks | Failure Condition |
|---|---|---|
| `source_health` | At least one source returned papers | All retrievers failed |
| `output_health` | Output files exist and are non-empty | Render phase produced nothing |
| `curation_size` | Post-filter paper count is reasonable | Too few or suspiciously many papers |
| `exclude_rate` | Suppression rate is within bounds | >90% of papers filtered out |

### Phase 14: Notification & Cleanup

- **Email**: If SMTP credentials are configured, a summary email is sent with
  paper counts, top papers, and a link to the repository
- **Snapshot pruning**: Snapshots beyond `retain_runs` (30) are deleted
- **Commit**: If `commit_state` and `commit_outputs_latest` are enabled,
  changed files are committed and pushed

---

## GitHub Actions Integration

The full lifecycle is orchestrated in `.github/workflows/daily.yml`:

```yaml
Schedule: cron(7 6 * * *)  # Daily at ~14:07 CST
Permissions: contents: write
Timeout: 30 minutes
Steps:
  1. Checkout repo
  2. Setup Python 3.12 + uv
  3. uv sync --frozen
  4. Readiness check
  5. Validate config
  6. Run pipeline (with API keys from secrets)
  7. Read commit policy from storage.yml
  8. Commit state + outputs (if policy allows)
  9. Send notification email
  10. Collect diagnostics → upload artifact
```

A separate workflow (`.github/workflows/validate.yml`) runs on every push and
PR: ruff lint, mypy type check, config validation, and pytest.
