# Workflow

## Daily Run Lifecycle

A LifeLit run is a single execution of `lifelit run --config config`. It
follows a strict phase order with no branching or parallelism at the pipeline
level (individual retrievers run sequentially for API rate-limit safety).

Authority note: this is supporting documentation. Current user-facing
operations are governed by `README.md` and `docs/user_guide.en.md`. Real
pipeline runs may call external services and write operating artifacts.

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

Before making any API calls, the pipeline verifies via offline deterministic
checks (no network calls, no API keys, no LLM access):

1. All 13 config files pass strict Pydantic validation
2. Retrieval strategy compiles without hard errors
3. At least one source is enabled
4. Commit policy and output hygiene are intentional

This is implemented in `readiness.py` and runs as a separate CI step in
`daily.yml` before the pipeline step. Use `uv run lifelit readiness --config
config` to run it locally.

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

Each enabled source is called sequentially. Failures are caught per-source in
the current implementation. `partial_failure_policy` validates but still needs
runtime-policy completion before docs can claim full fail/warn behavior.

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

Hardcoded rules run in priority order. Each rule can keep or suppress a paper.
Once suppressed by any rule, a paper is excluded from scoring and output
(except the suppressed CSV).

| # | Rule | Effect |
|---|---|---|
| 1 | Technical exclude | Drop papers missing title, ID, and authors |
| 2 | Author-tracked bypass | Author-tracked papers skip suppression |
| 3 | Manual-import bypass | Manually imported papers skip suppression |
| 4 | Hard suppress | Re-apply state-persisted suppression |
| 5 | Configured filter rules | Apply user-supplied filters (from filters.yml, not yet runtime-driving) |
| 6 | Publication type | Drop editorials, letters, retractions, etc. |
| 7 | Recommendation guardrails | Drop S2 recs with insufficient signal |
| 8 | Preprint guardrails | Drop preprints with no abstract and no venue |
| 9 | Metadata completeness | Drop papers missing both abstract and journal |
| 10 | Technical quality | Multi-dimension quality check |
| 11 | Default keep | All remaining papers pass through |

### Phase 8: Scoring

The 10-signal scoring engine computes per-paper scores within four sections:

| Section | Active Signals | Default Context |
|---|---|---|
| `author_tracked_manual` | entry, topic, metadata_quality, access, state | Higher base weight on entry_signal |
| `published` | entry, topic, journal, recency, metadata_quality, access, state | Higher weight on journal_signal |
| `preprinted` | entry, topic, journal, recency, metadata_quality, access, preprint_status, state | Preprint status signal active |
| `semantic_recommendations` | entry, topic, recommendation_profile, recommendation_rank, metadata_quality, access, state | Relies on recommendation signals |

Each signal returns a value in its registered range. The final score is a
**weighted sum** of per-signal values multiplied by per-section weights.
`scoring.yml` section overrides validate, but the active run path does not use
the config file's section overrides as the runtime scoring-context source.

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
6. **Fallback**: If the LLM is unavailable, a `fallback_tldr` from the LLM
   triage path provides a summary; the rule-based `fallback_tldr.py` extracts
   the first sentences of the abstract as a last resort.

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
4. **llm_cache.parquet** — new LLM responses are appended by the cache path.
   Privacy commit-policy flags require a future enforcement task before they
   can be treated as complete runtime controls.

### Phase 13: Health Gates

Four gates must pass for the run to be promoted to `outputs/latest/`:

| Gate | Checks | Failure Condition |
|---|---|---|
| `source_health` | At least one source returned papers | All attempted sources produced critical failures and zero records |
| `output_health` | Critical output files exist and are non-empty | `review_data.json` or `run_summary.json` missing |
| `curation_size` | Curated primary-surface paper count is reasonable | <10 or >200 papers |
| `exclude_rate` | Technical-exclude rate is within bounds | >50% of papers excluded |

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
