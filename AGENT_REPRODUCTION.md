# Agent Reproduction Guide

Authority note: this is supporting implementation guidance, not the current
source of truth. Current code registers 10 scoring signals and does not
register `penalty_signal`. Use `DESIGN.md`, `docs/user_guide.en.md`, and the
actual source files under `src/lifelit/` before using this guide for
implementation. The specifications below describe the intended architecture and
interfaces; the current implementation has evolved from these specs.

**Key divergences between this guide and current code (2026-05-27):**

- **Phase 2 models**: Actual model field names differ — `RawRecord` uses
  `source_record_id` (not `source_id`), `CanonicalPaper` uses `journal_name`
  (not `journal_normalized`) and has `entry_types` (plural, list) rather than
  `entry_type`, `SectionScoreResult` (not `ScoreResult`) carries `total_score`,
  `score_breakdown`, `matched_features`, and `llm_triage_selected` (not
  `llm_triage_eligible`).
- **Phase 4 normalization**: Produces `CandidatePaper` instances, not directly
  `CanonicalPaper`. Dedup merges `CandidatePaper`s into `CanonicalPaper`s.
- **Phase 5 scoring**: Per-topic saturation model (core binary 0.50, context K=2,
  support K=5, broad K=5) replaces global-flattened tier scoring. `entry_signal`
  maps `formal_retrieval` → 0.8 (not `formal_publication` → 0.7).
  `recency_signal` half-life is 365 days for published (not 30). Score
  aggregation is weighted sum, not weighted average. `recommendation_profile_signal`
  and `recommendation_rank_signal` are partially functional, not pure stubs.
- **Phase 7 state**: Uses PyArrow/pandas for Parquet, not polars.
- **Phase 8 CLI**: No `--verbose` flag. `run` command uses `--config`, `--mode`,
  `--newest-offset-days`, `--oldest-offset-days`. Source enrichment env var
  wiring passes through `enrich_all()` parameters, not hardcoded fallbacks.
- **Health gates**: `curation_size` thresholds are 10–200 papers, exclude rate
  threshold is 50% (not 90%).
- **Readiness checks**: Fully offline/deterministic — no network checks or env
  var checks.

This document is designed for **coding agents** (Claude Code, Cursor, Copilot,
etc.) to reproduce the LifeLit project from scratch. Each phase is
self-contained with explicit file paths, interfaces, and acceptance criteria.

Follow the phases in order. Each phase builds on the previous one. Run the
acceptance command at the end of each phase before proceeding.

---

## Phase 0: Project Scaffold

**Goal:** A Python package with dependencies, tooling, and a working CLI entry
point.

### Files to Create

```
pyproject.toml
.gitignore
src/lifelit/__init__.py
src/lifelit/__main__.py
src/lifelit/cli.py
```

### `pyproject.toml` Specification

```toml
[project]
name = "lifelit"
version = "0.1.0"
description = "Personal life-science literature monitor"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "click",
    "httpx",
    "pyarrow",
    "pydantic",
    "pyyaml",
    "streamlit",
]

[project.scripts]
lifelit = "lifelit.cli:main"

[dependency-groups]
dev = [
    "ruff",
    "mypy",
    "pytest",
    "pytest-cov",
    "types-pyyaml",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP"]

[tool.mypy]
python_version = "3.12"
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--basetemp=.pytest_tmp -m \"not integration\""
markers = [
    "integration: marks tests that require network access",
    "slow: marks tests that take more than 1 second",
]
```

### `src/lifelit/__init__.py`

```python
__version__ = "0.1.0"
```

### `src/lifelit/__main__.py`

```python
from lifelit.cli import main

if __name__ == "__main__":
    main()
```

### `src/lifelit/cli.py` (minimal skeleton)

```python
import click

@click.group()
@click.version_option()
def main() -> None:
    """LifeLit -- personal life-science literature monitor."""
    pass

@main.command()
@click.option("--config", "config_dir", default="config")
def run(config_dir: str) -> None:
    """Execute the full pipeline."""
    click.echo(f"Running with config from {config_dir}")

@main.command()
def review() -> None:
    """Launch review UI."""
    click.echo("Launching review UI...")

@main.command()
@click.option("--config", "config_dir", default="config")
def validate_config(config_dir: str) -> None:
    """Validate configuration files."""
    click.echo("Validating config...")

if __name__ == "__main__":
    main()
```

### Acceptance

```bash
uv sync
uv run lifelit --help
uv run lifelit version
```

---

## Phase 1: Configuration System

**Goal:** Pydantic v2 models for all 13 config files, YAML loading, and
cross-file validation.

### Files to Create

```
src/lifelit/config.py
config/sources.yml
config/runtime.yml
config/output.yml
config/filters.yml
config/privacy.yml
config/llm.yml
config/storage.yml
config/seeds.yml
config/scoring.yml
config/journals.yml
config/authors.yml
config/topics.yml
config/semantic_scholar.yml
tests/test_config.py
```

### `config.py` Architecture

Create an `AppConfig` model that loads and validates all 13 sub-configs:

```python
from pydantic import BaseModel, Field

class AppConfig(BaseModel):
    sources: SourcesConfig
    runtime: RuntimeConfig
    output: OutputConfig
    filters: FiltersConfig
    privacy: PrivacyConfig
    llm: LlmConfig
    storage: StorageConfig
    seeds: SeedsConfig
    scoring: ScoringConfig
    journals: JournalsConfig
    authors: AuthorsConfig
    topics: TopicsConfig
    semantic_scholar: SemanticScholarConfig

    @classmethod
    def from_directory(cls, config_dir: str) -> "AppConfig":
        """Load and validate all YAML files from a directory."""
        ...
```

### Key Model Specifications

**SourcesConfig** — Map of source name to `SourceEntry`:
- `SourceEntry`: `enabled: bool`, `roles: list[str]`, plus source-specific
  optional fields (`api_key_env`, `daily_date_type`, `backfill_date_type`,
  `date_field`, `mailto_env`, `user_agent_env`)

**RuntimeConfig**:
- `mode: str` (allowed: `daily`, `dry_run`, `backfill`, `manual`)
- `timezone: str` (any IANA timezone string)
- `newest_offset_days: int` (>= 0)
- `oldest_offset_days: int` (>= 1)
- `partial_failure_policy: str` (allowed: `warn`, `fail`)

**OutputConfig** — Seven boolean toggles for output formats.

**FiltersConfig** — Contains `filters: list[FilterRule]`. `FilterRule` has
`target` (paper field), `operator` (contains, equals, missing, etc.), `value`,
and `action` (suppress, flag, boost).

**PrivacyConfig** — Four boolean fields: `commit_llm_cache`,
`commit_raw_llm_prompts`, `commit_raw_api_payloads`,
`include_abstracts_in_outputs`.

**LlmConfig** — Provider config (env var names for base_url, api_key, model),
`eligible_sections: list[str]`, `max_papers_per_run: int`,
`max_papers_per_section: int`, `max_score_adjustment: float`,
`BudgetConfig` (per-run/per-paper token and cost limits, cost rates),
`CacheConfig` (enabled + path), `FallbackTldrConfig` (enabled + max_length).

**StorageConfig** — `state_dir`, `output_dir`, `feedback_events_dir`,
`commit_state`, `commit_outputs_latest`, `commit_output_snapshots`,
`retain_runs`, `atomic_writes`.

**SeedsConfig** — `positive_seeds: list[str]`, `negative_seeds: list[str]`,
`boundary_seeds: list[str]`, `seed_profiles: list[SeedProfile]`.
`SeedProfile` has `id`, `label`, `description`, `enabled`,
`recommendation_limit`, `positive_seeds`, `negative_seeds`, `boundary_seeds`.

**ScoringConfig** — `sections: list[str]`, `signals: list[str]`,
`section_overrides: dict[str, SectionOverride]`.
`SectionOverride` has `signal_weights: dict[str, float]` and
`config: dict[str, Any]`. Cross-validate that signal names in overrides exist
in the `signals` list.

**JournalsConfig** — `journals: list[JournalEntry]`.
`JournalEntry`: `name`, `aliases: list[str]`, `issn: list[str]`,
`eissn: list[str]`, `roles: list[str]` (allowed: `normalize`, `reputation`,
`formal_retrieval`), `reputation_tier: str | None` (allowed: `top`, `high`,
`standard`), `openalex_source_id: str | None`, `nlm_abbreviation: str | None`,
`clinical_trial_high_level: bool`, `journal_signal_coefficient: float | None`
(validated to [0.714286, 1.428571]).

**AuthorsConfig** — `authors: list[AuthorEntry]`.
`AuthorEntry`: `author_id: str`, `author_id_type: str` (allowed: `orcid`).

**TopicsConfig** — `topics: list[TopicEntry]`.
`TopicEntry`: `name`, `id`, `label`, `retrieval_logic`
(allowed: `MESH_OR_KEYWORDS`, `MESH_ONLY`, `KEYWORDS_ONLY`), `priority`
(allowed: `strong`, `secondary`), `intent`, plus many list[str] fields for
keywords, MeSH terms, scoring terms, and tiered terms (core, context, broad,
support, boundary).

**SemanticScholarConfig** — `recommendation_profiles:
list[RecommendationProfile]`, `recommendation_limit: int`,
`standard_enrichment_fields: list[str]`, `deep_enrichment_top_n: int`,
`search_enabled_for_daily: bool`. `RecommendationProfile` has `id`, `label`,
`description`, `enabled`, `limit`, `weight`, `positive_seed_ids`,
`negative_seed_ids`, `positive_seeds`, `negative_seeds`.

### CLI Commands to Implement

```bash
lifelit validate-config --config config  # validates all 13 files
```

### Acceptance

```bash
uv run lifelit validate-config --config config
# Should pass with valid config, fail with descriptive errors on invalid config
pytest tests/test_config.py -v
```

---

## Phase 2: Domain Models

**Goal:** Core data classes that flow through the pipeline.

### Files to Create

```
src/lifelit/models.py
tests/test_models.py
```

### Models to Define

```python
@dataclass
class RawRecord:
    """A paper record from a single source, before normalization."""
    source: str
    source_id: str
    title: str
    authors: list[str]
    abstract: str
    doi: str | None
    pmid: str | None
    pmcid: str | None
    journal: str
    issn: str | None
    eissn: str | None
    date: date | None
    entry_type: str  # "author_tracked", "formal_publication", "preprint",
                      # "semantic_recommendation", "manual_import"
    raw_payload: dict[str, Any]

@dataclass
class CanonicalPaper:
    """Normalized, deduplicated paper ready for enrichment and scoring."""
    dedup_key: str
    title: str
    authors: list[str]
    abstract: str
    doi: str | None
    pmid: str | None
    pmcid: str | None
    journal_normalized: str
    issn: str | None
    eissn: str | None
    date: date | None
    entry_type: str
    source_ids: list[str]  # all source IDs merged during dedup
    sources: list[str]     # all source names
    # Enrichment fields (populated later)
    tldr: str = ""
    citation_count: int = 0
    is_open_access: bool = False
    published_version_doi: str | None = None
    preprint_doi: str | None = None

@dataclass
class ScoreResult:
    """Per-paper scoring output from the 10-signal engine."""
    dedup_key: str
    section: str
    final_score: float
    score_band: str  # "high", "medium", "low"
    signal_values: dict[str, float]
    signal_weights: dict[str, float]
    signal_evidence: dict[str, dict[str, Any]]
    deep_enrichment_eligible: bool = False
    deep_enrichment_reason: str = ""
    llm_triage_eligible: bool = False
    llm_triage_reason: str = ""

@dataclass
class LlmTriageResult:
    """LLM triage output for a single paper."""
    dedup_key: str
    score_adjustment: float
    adjusted_score: float
    tldr: str
    rationale: str
    model: str
    tokens_used: int
    from_cache: bool

@dataclass
class ClinicalTrialAnnotation:
    """Clinical trial observation annotation."""
    dedup_key: str
    is_clinical_trial: bool
    matched_venue: str
    match_method: str
```

### Acceptance

```bash
pytest tests/test_models.py -v
```

---

## Phase 3: Retrieval Layer

**Goal:** Five API clients + retrieval strategy compiler.

### Files to Create

```
src/lifelit/retrieval_strategy.py
src/lifelit/pubmed.py
src/lifelit/europe_pmc.py
src/lifelit/biorxiv.py
src/lifelit/semantic_scholar.py
src/lifelit/author_tracked.py
src/lifelit/retrieval.py
tests/test_pubmed.py
tests/test_europe_pmc.py
tests/test_biorxiv.py
tests/test_semantic_scholar.py
tests/test_author_tracked.py
tests/test_retrieval_strategy.py
```

### `retrieval_strategy.py` — Strategy Compiler

Compiles config into per-source dataclass instances:

```python
@dataclass
class PubmedCompiledStrategy:
    source_role: str
    topic_clauses: list[str]
    journal_filter: list[str] | None
    date_type: str
    newest_offset_days: int
    oldest_offset_days: int

@dataclass
class EuropePMCCompiledStrategy:
    source_role: str
    topic_clauses: list[str]
    journal_filter: list[str] | None
    newest_offset_days: int
    oldest_offset_days: int

@dataclass
class BiorxivCompiledStrategy:
    source_role: str
    filter_terms: list[str]
    date_field: str
    newest_offset_days: int
    oldest_offset_days: int

@dataclass
class SemanticScholarCompiledStrategy:
    source_role: str
    positive_seeds: list[str]
    negative_seeds: list[str]
    boundary_seeds: list[str]
    limit: int

def compile_topics(topics: list[TopicEntry]) -> list[CompiledTopic]:
    """Extract query terms and scoring terms from topic config."""
    ...

def compile_pubmed(topics, journals, config) -> PubmedCompiledStrategy | None:
    ...

def compile_europe_pmc(topics, journals, config) -> EuropePMCCompiledStrategy | None:
    ...

def compile_biorxiv(topics, config) -> BiorxivCompiledStrategy | None:
    ...

def compile_semantic_scholar(seeds, config) -> SemanticScholarCompiledStrategy | None:
    ...
```

### API Client Specifications

**PubMed** (`pubmed.py`):
- `retrieve_pubmed(topic_clauses, journal_filter, date_type, newest_offset,
  oldest_offset, api_key) -> list[RawRecord]`
- Uses E-Utilities: esearch → efetch (batch)
- Constructs query: `(topic_clause_1 OR topic_clause_2 OR ...) AND
  (journal_filter_1 OR ...) AND (date_range)`
- Date range: `("YYYY/MM/DD"[date_type] : "YYYY/MM/DD"[date_type])`
- Parses XML response into `RawRecord` with `entry_type="formal_publication"`

**Europe PMC** (`europe_pmc.py`):
- `retrieve_europe_pmc(topic_clauses, journal_filter, newest_offset,
  oldest_offset) -> list[RawRecord]`
- REST API: `https://www.ebi.ac.uk/europepmc/webservices/rest/search`
- Query: `(TITLE_ABSTRACT:"term1" OR TITLE_ABSTRACT:"term2") AND
  (FIRST_PDATE:[date1 TO date2])`
- Parses JSON response into `RawRecord` with `entry_type="formal_publication"`

**bioRxiv** (`biorxiv.py`):
- `retrieve_biorxiv(filter_terms, date_field, newest_offset, oldest_offset)
  -> list[RawRecord]`
- API: `https://api.biorxiv.org/details/biorxiv/YYYY-MM-DD/YYYY-MM-DD/0`
- Post-retrieval filter: title + abstract must contain at least one filter term
- `entry_type="preprint"`

**Semantic Scholar** (`semantic_scholar.py`):
- `retrieve_semantic_scholar_recommendations(positive_seeds, negative_seeds,
  api_key, limit, profile_id=None) -> list[RawRecord]`
- API: `https://api.semanticscholar.org/recommendations/v1/papers`
- Input: list of positive DOIs, optional negative DOIs
- `entry_type="semantic_recommendation"`
- `retrieve_semantic_scholar_search(query, api_key, limit) -> list[RawRecord]`
  (for keyword search fallback)

**Author Tracked** (`author_tracked.py`):
- `retrieve_author_tracked(authors, newest_offset_days, oldest_offset_days)
  -> list[RawRecord]`
- For each ORCID: resolve to S2 author ID, fetch recent papers
- Filter by date window
- `entry_type="author_tracked"`

### Acceptance

```bash
pytest tests/test_pubmed.py tests/test_europe_pmc.py -v
pytest tests/test_biorxiv.py tests/test_semantic_scholar.py -v
pytest tests/test_retrieval_strategy.py -v
# Integration tests (require network):
pytest -m integration tests/test_pubmed.py -v
```

---

## Phase 4: Processing Pipeline

**Goal:** Normalization → Deduplication → Enrichment → Filtering.

### Files to Create

```
src/lifelit/normalization.py
src/lifelit/dedup.py
src/lifelit/enrichment.py
src/lifelit/filtering.py
tests/test_normalization.py
tests/test_dedup.py
tests/test_enrichment.py
tests/test_filtering.py
```

### `normalization.py`

```python
def normalize_journal_name(
    raw_journal: str, issn: str | None, eissn: str | None,
    journals: list[JournalEntry],
) -> str:
    """Match against journal name, aliases, ISSN, eISSN. Return canonical name."""
    ...

def normalize_raw_records(
    records: list[RawRecord], journals: list[JournalEntry],
) -> list[CanonicalPaper]:
    """Convert RawRecord → CanonicalPaper with normalized journal names."""
    ...
```

### `dedup.py`

```python
def deduplicate(papers: list[CanonicalPaper]) -> list[CanonicalPaper]:
    """Deduplicate by DOI > PMID > title hash.
    When merging, keep the richest metadata and merge source_ids."""
    ...

def compute_dedup_key(paper: CanonicalPaper) -> str:
    """DOI if present, else PMID, else normalized title hash."""
    ...
```

### `enrichment.py`

```python
async def enrich_semantic_scholar(
    papers: list[CanonicalPaper], api_key: str | None, fields: list[str] | None,
) -> None:
    """Fetch TLDRs and citation counts from S2 API. Mutates papers in place."""
    ...

async def enrich_openalex(
    papers: list[CanonicalPaper], api_key: str | None,
) -> None:
    """Fetch OA status from OpenAlex. Mutates papers in place."""
    ...

async def enrich_crossref(
    papers: list[CanonicalPaper], mailto_env: str,
) -> None:
    """Fetch published version DOI and metadata from Crossref."""
    ...

async def enrich_all(papers: list[CanonicalPaper]) -> None:
    """Run all enrichment steps. Failures are logged, not fatal."""
    ...
```

### `filtering.py`

Implement 9 hardcoded rules in order:

```python
def apply_filters(
    papers: list[CanonicalPaper],
    previous_decisions: dict[str, str] | None = None,
) -> tuple[list[CanonicalPaper], list[CanonicalPaper]]:
    """Returns (kept, suppressed). Rules run in order; first match wins."""
    ...

# Individual rules:
def _rule_technical_exclude(paper) -> bool  # missing title + id + authors
def _rule_author_tracked_bypass(paper) -> bool  # author-tracked bypass
def _rule_manual_import_bypass(paper) -> bool  # manual import bypass
def _rule_hard_suppress(paper, previous_decisions) -> bool  # persisted state
def _rule_publication_type(paper) -> bool  # editorial, letter, retraction
def _rule_recommendation_guardrails(paper) -> bool  # insufficient S2 signal
def _rule_preprint_guardrails(paper) -> bool  # no abstract + no venue
def _rule_metadata_completeness(paper) -> bool  # missing abstract + journal
def _rule_technical_quality(paper) -> bool  # multi-dimension quality
```

### Acceptance

```bash
pytest tests/test_normalization.py tests/test_dedup.py -v
pytest tests/test_enrichment.py tests/test_filtering.py -v
```

---

## Phase 5: Scoring Engine

**Goal:** 10-signal multi-factor scoring with per-section weights.

### Files to Create

```
src/lifelit/scoring.py
src/lifelit/clinical.py
tests/test_scoring.py
tests/test_clinical.py
```

### `scoring.py` Architecture

```python
# Signal definitions
@dataclass
class SignalDefinition:
    name: str
    weight: float
    range: tuple[float, float]  # always (0.0, 1.0)
    description: str

REGISTERED_SIGNALS: dict[str, SignalDefinition] = {
    "entry_signal": SignalDefinition("entry_signal", 1.0, (0.0, 1.0), "..."),
    "topic_signal": SignalDefinition("topic_signal", 1.0, (0.0, 1.0), "..."),
    "journal_signal": SignalDefinition("journal_signal", 1.0, (0.0, 1.0), "..."),
    "recommendation_profile_signal": SignalDefinition(..., 0.5, ...),
    "recommendation_rank_signal": SignalDefinition(..., 0.5, ...),
    "recency_signal": SignalDefinition(..., 1.0, ...),
    "metadata_quality_signal": SignalDefinition(..., 0.5, ...),
    "access_signal": SignalDefinition(..., 0.5, ...),
    "preprint_status_signal": SignalDefinition(..., 0.5, ...),
    "state_signal": SignalDefinition(..., 0.0, ...),
}

# Section definitions with preset contexts
@dataclass
class SectionContext:
    section_name: str
    active_signals: list[str]
    signal_weights: dict[str, float]
    config: dict[str, Any]  # extra params for signal computers

PRESET_CONTEXTS: dict[str, SectionContext] = {
    "author_tracked_manual": SectionContext(
        active_signals=["entry_signal", "topic_signal", "recency_signal",
                        "metadata_quality_signal", "access_signal",
                        "preprint_status_signal", "state_signal"],
        signal_weights={"entry_signal": 2.0, ...},
        config={},
    ),
    "published": SectionContext(...),
    "preprinted": SectionContext(...),
    "semantic_recommendations": SectionContext(...),
}

# Signal computer functions
def _compute_entry_signal(paper, context) -> float: ...
def _compute_topic_signal(paper, context) -> float: ...
def _compute_journal_signal(paper, context) -> float: ...
# ... all 10 registered signals ...

_SIGNAL_COMPUTERS: dict[str, Callable] = {...}

# Main scoring functions
def compute_section_score(
    paper: CanonicalPaper, context: SectionContext,
) -> ScoreResult: ...

def compute_all_scores(
    papers: list[CanonicalPaper],
    section_contexts: dict[str, SectionContext] | None = None,
) -> list[ScoreResult]: ...

def build_journal_reputation_lookup(
    journals: list[JournalEntry],
) -> list[dict[str, Any]]: ...

def get_all_section_contexts() -> dict[str, SectionContext]:
    return dict(PRESET_CONTEXTS)

def merge_context_with_overrides(
    context: SectionContext, config_overrides: dict[str, object],
) -> SectionContext: ...
```

### Signal Details

**entry_signal**: Maps `entry_type` to base score.
- `author_tracked` → 1.0, `manual_import` → 0.9, `formal_publication` → 0.7,
  `preprint` → 0.5, `semantic_recommendation` → 0.5

**topic_signal**: Matches tiered scoring terms against title + abstract.
- core_terms match → 1.0, context_terms → 0.8, broad_terms → 0.5,
  support_terms → 0.3, boundary_terms → 0.1
- Falls back to flat `scoring_terms` if no tiered terms are configured

**journal_signal**: ISSN/name/alias match against reputation tiers.
- `top` → 1.0, `high` → modulated by `journal_signal_coefficient` to
  [0.5, 1.0], `standard` → 0.3, unranked → 0.0

**recency_signal**: Exponential decay: `exp(-days_since_publication / half_life)`.
Half-life defaults to 30 days.

**metadata_quality_signal**: Fraction of 7 metadata fields that are present
(doi, pmid, abstract, title, journal, authors, date).

**access_signal**: Checks Europe PMC and OpenAlex OA status flags.

**preprint_status_signal**: 1.0 if preprint has a published version DOI, 0.5
otherwise.

**recommendation_profile_signal, recommendation_rank_signal, state_signal**:
Currently return neutral defaults (0.5 or 0.0). Designed for future
implementation.

There is no registered `penalty_signal` in the current source. Penalty-like
effects belong in rule/scoring adjustments, warnings, or score-result metadata.

### `clinical.py`

```python
def annotate_clinical_trials(
    papers: list[CanonicalPaper],
    journals: list[JournalEntry],
) -> dict[str, ClinicalTrialAnnotation]:
    """Check papers against journals with clinical_trial_high_level=True.
    Match by journal name, alias, ISSN, eISSN."""
    ...
```

### Acceptance

```bash
pytest tests/test_scoring.py -v
pytest tests/test_clinical.py -v
# Verify all 10 signals are registered and have computer functions
```

---

## Phase 6: LLM Integration

**Goal:** OpenAI-compatible LLM client, prompt templates, budget control,
response cache, and fallback TLDR.

### Files to Create

```
src/lifelit/llm_client.py
src/lifelit/prompts.py
src/lifelit/budget.py
src/lifelit/cache.py
src/lifelit/fallback_tldr.py
src/lifelit/llm_triage.py
tests/test_llm_client.py
tests/test_budget.py
tests/test_cache.py
tests/test_fallback_tldr.py
tests/test_llm_triage.py
```

### `llm_client.py`

```python
@dataclass
class LlmClientConfig:
    base_url: str
    api_key: str
    model: str

class LlmClient:
    def __init__(self, config: LlmClientConfig): ...
    async def complete(self, prompt: str, max_tokens: int) -> str: ...
    # Wraps httpx to call OpenAI-compatible /v1/chat/completions
```

### `prompts.py`

```python
SCORING_SYSTEM_PROMPT = """You are a life-science literature triage assistant..."""

def build_scoring_prompt(
    paper: CanonicalPaper, context_terms: list[str] | None,
) -> str:
    """Build a structured prompt asking for score adjustment and TLDR."""
    ...

def parse_scoring_response(response: str) -> dict[str, Any]:
    """Parse JSON from LLM response. Returns {score_adjustment, tldr, rationale}."""
    ...
```

### `budget.py`

```python
class BudgetManager:
    """Tracks token usage and cost against per-run and per-paper limits."""
    def __init__(self, config: BudgetConfig): ...
    def can_afford(self, estimated_input_tokens: int, estimated_output_tokens: int) -> bool: ...
    def record_usage(self, input_tokens: int, output_tokens: int) -> None: ...
    @property
    def total_tokens(self) -> int: ...
    @property
    def total_cost(self) -> float: ...
```

### `cache.py`

```python
class LlmCache:
    """Parquet-backed cache keyed by SHA-256 prompt hash."""
    def __init__(self, path: str): ...
    def get(self, prompt: str) -> dict[str, Any] | None: ...
    def set(self, prompt: str, response: dict[str, Any], model: str, tokens: int) -> None: ...
```

### `fallback_tldr.py`

```python
def extract_fallback_tldr(abstract: str, max_length: int = 280) -> str:
    """Extract first 1-2 sentences of abstract as TLDR."""
    ...
```

### `llm_triage.py`

```python
async def run_llm_triage(
    papers: list[tuple[CanonicalPaper, ScoreResult]],
    client: LlmClient,
    budget: BudgetManager,
    cache: LlmCache,
    config: LlmConfig,
    context_terms: list[str] | None = None,
) -> tuple[list[LlmTriageResult], BudgetManager, int]:
    """Triage eligible papers through LLM. Returns (results, budget, cache_hits)."""
    ...
```

### Acceptance

```bash
pytest tests/test_budget.py tests/test_cache.py -v
pytest tests/test_fallback_tldr.py tests/test_llm_triage.py -v
```

---

## Phase 7: Output, State, and Notification

**Goal:** Six output renderers, Parquet state management, and email
notification.

### Files to Create

```
src/lifelit/render.py
src/lifelit/state/__init__.py
src/lifelit/state/suppression.py
src/lifelit/notify.py
src/lifelit/run_health.py
src/lifelit/source_reliability.py
tests/test_render.py
tests/test_state.py
tests/test_notify.py
tests/test_run_health.py
tests/test_source_reliability.py
```

### `render.py` — Output Renderers

Each renderer takes review data (papers + scores + triage + annotations) and
writes a file:

```python
def build_review_data(...) -> dict[str, Any]:
    """Assemble all pipeline outputs into a unified review data dict."""
    ...

def write_review_data_json(data: dict, path: str) -> None: ...

def render_report_md_from_review_data(data: dict) -> str:
    """Generate human-readable Markdown briefing:
    # LifeLit Daily Briefing
    ## Section: Published (High)
    ### Paper Title
    **Score:** 0.85 | **Journal:** Nature
    TLDR: ...
    """
    ...

def write_report_md(markdown: str, path: str) -> None: ...

def render_papers_csv_from_review_data(data: dict) -> str: ...
def write_papers_csv(csv_content: str, path: str) -> None: ...

def render_zotero_csv_from_review_data(data: dict) -> str: ...
def write_zotero_csv(csv_content: str, path: str) -> None: ...

def render_suppressed_csv(suppressed: list[CanonicalPaper]) -> str: ...
def write_suppressed_csv(csv_content: str, path: str) -> None: ...

def render_run_summary(...) -> dict[str, Any]: ...
def write_run_summary_json(data: dict, path: str) -> None: ...
```

Note: `static_html` is not implemented; the config toggle exists but has no
backing renderer.

### `state/__init__.py` — Parquet State Management

```python
def read_seen_papers(state_dir: str) -> pl.DataFrame | None: ...
def write_seen_papers(df: pl.DataFrame, state_dir: str) -> None: ...
def update_seen_papers(new_papers: list[CanonicalPaper], run_id: str, state_dir: str) -> pl.DataFrame: ...

def read_paper_status(state_dir: str) -> pl.DataFrame | None: ...
def write_paper_status(df: pl.DataFrame, state_dir: str) -> None: ...

def read_run_index(state_dir: str) -> pl.DataFrame | None: ...
def append_run_index(entry: dict, state_dir: str) -> None: ...

def read_llm_cache(cache_path: str) -> pl.DataFrame | None: ...
def write_llm_cache(df: pl.DataFrame, cache_path: str) -> None: ...

def _atomic_write_parquet(df: pl.DataFrame, path: str) -> None:
    """Write to temp file then rename for atomicity."""
    ...
```

### `notify.py`

```python
def send_email_notification(
    smtp_host: str, smtp_port: int, username: str, password: str,
    to_addr: str, from_addr: str, subject: str, body: str,
) -> None: ...

def build_notification_body(run_summary: dict, top_papers: list) -> str:
    """Build plain-text email body with paper counts and highlights."""
    ...
```

### `run_health.py`

```python
@dataclass
class HealthResult:
    passed: bool
    gate: str
    details: str

def check_source_health(source_counts: dict[str, int]) -> HealthResult: ...
def check_output_health(output_dir: str, output_config: OutputConfig) -> HealthResult: ...
def check_curation_size(kept: int, suppressed: int) -> HealthResult: ...
def check_exclude_rate(kept: int, suppressed: int) -> HealthResult: ...

def should_promote(
    source_counts: dict, output_dir: str, output_config: OutputConfig,
    kept: int, suppressed: int,
) -> tuple[bool, list[HealthResult]]:
    """Run all gates. Return (promote, results)."""
    ...
```

### `source_reliability.py`

```python
@dataclass
class SourceFailure:
    source: str
    operation: str
    exception_type: str
    message: str
    timestamp: str

def source_failure_from_exception(
    exc: Exception, source: str, operation: str,
) -> SourceFailure: ...

def source_failure_to_dict(failure: SourceFailure) -> dict[str, Any]: ...
```

### Acceptance

```bash
pytest tests/test_render.py tests/test_state.py -v
pytest tests/test_notify.py tests/test_run_health.py -v
pytest tests/test_source_reliability.py -v
```

---

## Phase 8: CLI Orchestrator, Review UI, and CI/CD

**Goal:** Wire all phases together in `cli.py`, add Streamlit review UI, and
configure GitHub Actions.

### Files to Update/Create

```
src/lifelit/cli.py (full orchestrator)
src/lifelit/review_ui.py
src/lifelit/readiness.py
.github/workflows/daily.yml
.github/workflows/validate.yml
tests/test_cli.py
```

### `cli.py` — Full Pipeline Orchestrator

The `run` command executes all phases in sequence:

```python
@main.command()
@click.option("--config", "config_dir", default="config")
@click.option("--mode", "run_mode_override", default=None)
@click.option("--newest-offset", type=int, default=None)
@click.option("--oldest-offset", type=int, default=None)
@click.option("--verbose", is_flag=True)
def run(config_dir, run_mode_override, newest_offset, oldest_offset, verbose):
    """Execute the full LifeLit pipeline."""
    config = AppConfig.from_directory(config_dir)

    # Phase 0: Validate
    # Phase 1: Compile retrieval strategy
    # Phase 2: Retrieve from each enabled source
    # Phase 3: Normalize
    # Phase 4: Deduplicate
    # Phase 5: Enrich
    # Phase 6: Filter
    # Phase 7: Score (build contexts with topic terms + journal reputation)
    # Phase 8: Clinical trial annotation
    # Phase 9: LLM triage (if enabled)
    # Phase 10: Render outputs
    # Phase 11: Update state
    # Phase 12: Health gates → promote if healthy
    # Phase 13: Prune old snapshots
    ...
```

Key wiring details:

1. Topic terms from `compile_topics()` are injected into scoring contexts via
   `merge_context_with_overrides()`
2. Journal reputation from `build_journal_reputation_lookup()` is injected into
   published and preprinted section contexts
3. `enrich_all()` is called with no keyword arguments (config values for API
   keys are resolved from environment variables within each enrichment
   function)
4. Source failures are caught per-source and tracked via
   `source_failure_from_exception()` / `source_failure_to_dict()`
5. Output rendering checks each `config.output.outputs.*` boolean before
   calling the corresponding render function
6. State updates happen after rendering (so rendered output reflects pre-state
   data)
7. Health gates run on the completed output; promotion copies snapshot to
   `outputs/latest/`

### `review_ui.py` — Streamlit Review UI

```python
# Streamlit app for browsing and curating papers
# Pages: Paper List (filterable/sortable), Paper Detail, Feedback History
# Actions: star, suppress, dismiss (saved as YAML feedback events)
```

### `readiness.py` — Offline Readiness Checks

```python
def check_env_vars(required_vars: list[str]) -> list[str]:
    """Check that required environment variables are set."""
    ...

def check_network() -> bool:
    """Verify network connectivity."""
    ...

def check_state_files(state_dir: str) -> bool:
    """Verify state Parquet files exist or can be created."""
    ...
```

### GitHub Actions Workflows

**`daily.yml`** — Scheduled pipeline run:
- Trigger: `schedule: cron(7 6 * * *)` and `workflow_dispatch`
- Permission: `contents: write`
- Timeout: 30 minutes
- Steps: checkout → setup Python 3.12 → setup uv → `uv sync --frozen` →
  readiness check → validate config → run pipeline → commit state + outputs →
  send notification → upload diagnostics artifact

**`validate.yml`** — CI on push/PR:
- Trigger: `push` and `pull_request` to `main`
- Permission: `contents: read`
- Timeout: 10 minutes
- Steps: checkout → setup → lint (ruff) → type check (mypy strict) → validate
  config → pytest

### Additional CLI Commands to Implement

```
lifelit review                  # Launch Streamlit UI
lifelit validate-config         # Validate all config files
lifelit readiness               # Run offline readiness checks
lifelit notify                  # Send email notification
lifelit profile compile         # NL profile → config draft
lifelit profile validate        # Validate profile-generated draft
lifelit profile diff            # Show draft changes
lifelit feedback apply          # Apply feedback events
lifelit feedback list           # List feedback events
lifelit feedback undo           # Undo a feedback event
lifelit feedback propose        # Generate config proposals
lifelit feedback config-draft   # Generate config draft from proposals
lifelit config-promote          # Promote draft to live config
lifelit retrieval simulate      # Simulate retrieval without network
lifelit import-manual           # Import manual paper records
lifelit version                 # Show version
```

### Acceptance

```bash
# Full pipeline test
uv run lifelit run --config config --mode dry_run
uv run lifelit validate-config --config config
uv run lifelit readiness --config config

# Type check and lint
uv run mypy src
uv run ruff check src

# Test suite
uv run pytest -m "not integration" -v
```

---

## Reproduction Checklist

After completing all 8 phases, verify the full system:

- [ ] `uv sync` installs all dependencies without errors
- [ ] `uv run lifelit --help` shows all 19 commands
- [ ] `uv run lifelit validate-config --config config` passes
- [ ] `uv run lifelit run --config config --mode dry_run` compiles strategy
- [ ] `uv run lifelit run --config config` completes a full local run
- [ ] `uv run mypy src` passes in strict mode
- [ ] `uv run ruff check src` passes
- [ ] `uv run pytest -m "not integration" -v` passes
- [ ] `outputs/latest/review_data.json` exists and is valid JSON
- [ ] `outputs/latest/report.md` exists and is human-readable
- [ ] `state/seen_papers.parquet` exists and is valid Parquet
- [ ] `state/run_index.parquet` contains the run record
- [ ] GitHub Actions `daily.yml` runs successfully on schedule
- [ ] GitHub Actions `validate.yml` passes on push

## Dependency Order Summary

```
Phase 0: scaffold (no dependencies)
Phase 1: config.py (no dependencies)
Phase 2: models.py (no dependencies)
Phase 3: retrieval (depends on Phase 1, 2)
Phase 4: processing (depends on Phase 2, 3)
Phase 5: scoring (depends on Phase 2, 4)
Phase 6: LLM (depends on Phase 2, 5)
Phase 7: output + state (depends on Phase 2, 5, 6)
Phase 8: CLI + CI/CD (depends on all previous phases)
```
