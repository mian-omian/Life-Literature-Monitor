# Data Model

## Core Domain Models

```mermaid
classDiagram
    class RawRecord {
        +str source
        +str source_id
        +str title
        +list~str~ authors
        +str abstract
        +str doi
        +str pmid
        +str pmcid
        +str journal
        +str issn
        +str eissn
        +date date
        +str entry_type
        +dict raw_payload
    }

    class CanonicalPaper {
        +str dedup_key
        +str title
        +list~str~ authors
        +str abstract
        +str doi
        +str pmid
        +str pmcid
        +str journal_normalized
        +str issn
        +str eissn
        +date date
        +str entry_type
        +list~str~ source_ids
        +list~str~ sources
        +str tldr
        +int citation_count
        +bool is_open_access
        +str published_version_doi
        +str preprint_doi
    }

    class ScoreResult {
        +str dedup_key
        +str section
        +float final_score
        +str score_band
        +dict~str,float~ signal_values
        +dict~str,float~ signal_weights
        +dict~str,dict~ signal_evidence
        +bool deep_enrichment_eligible
        +str deep_enrichment_reason
        +bool llm_triage_eligible
        +str llm_triage_reason
    }

    class LlmTriageResult {
        +str dedup_key
        +float score_adjustment
        +float adjusted_score
        +str tldr
        +str rationale
        +str model
        +int tokens_used
        +bool from_cache
    }

    class ClinicalTrialAnnotation {
        +str dedup_key
        +bool is_clinical_trial
        +str matched_venue
        +str match_method
    }

    class SeenPaper {
        +str dedup_key
        +str first_seen_run
        +str last_seen_run
        +list~str~ source_ids
    }

    class PaperStatus {
        +str dedup_key
        +str status
        +str suppressed_reason
        +datetime updated_at
    }

    class RunIndexEntry {
        +str run_id
        +datetime timestamp
        +str mode
        +int paper_count
        +bool health_passed
        +dict diagnostics
    }

    CanonicalPaper "1" --> "*" RawRecord : normalized from
    ScoreResult "1" --> "1" CanonicalPaper : scores
    LlmTriageResult "1" --> "1" ScoreResult : triages
    ClinicalTrialAnnotation "1" --> "1" CanonicalPaper : annotates
    SeenPaper "1" --> "1" CanonicalPaper : tracks
    PaperStatus "1" --> "1" CanonicalPaper : curates
    RunIndexEntry "1" --> "*" CanonicalPaper : indexes
```

## Config Model Hierarchy

```mermaid
classDiagram
    class AppConfig {
        +SourcesConfig sources
        +RuntimeConfig runtime
        +OutputConfig output
        +FiltersConfig filters
        +PrivacyConfig privacy
        +LlmConfig llm
        +StorageConfig storage
        +SeedsConfig seeds
        +ScoringConfig scoring
        +JournalsConfig journals
        +AuthorsConfig authors
        +TopicsConfig topics
        +SemanticScholarConfig semantic_scholar
    }

    class SourcesConfig {
        +SourceEntry pubmed
        +SourceEntry europe_pmc
        +SourceEntry biorxiv
        +SourceEntry semantic_scholar
        +SourceEntry openalex
        +SourceEntry crossref
    }

    class ScoringConfig {
        +list~str~ sections
        +list~str~ signals
        +dict~str,SectionOverride~ section_overrides
    }

    class JournalsConfig {
        +list~JournalEntry~ journals
    }

    class TopicsConfig {
        +list~TopicEntry~ topics
    }

    class SeedsConfig {
        +list~str~ positive_seeds
        +list~str~ negative_seeds
        +list~str~ boundary_seeds
        +list~SeedProfile~ seed_profiles
    }

    class LlmConfig {
        +ProviderConfig provider
        +list~str~ eligible_sections
        +int max_papers_per_run
        +int max_papers_per_section
        +float max_score_adjustment
        +BudgetConfig budget
        +CacheConfig cache
        +FallbackTldrConfig fallback_tldr
    }

    AppConfig *-- SourcesConfig
    AppConfig *-- ScoringConfig
    AppConfig *-- JournalsConfig
    AppConfig *-- TopicsConfig
    AppConfig *-- SeedsConfig
    AppConfig *-- LlmConfig
```

## State File Schemas

### seen_papers.parquet
| Column | Type | Description |
|---|---|---|
| `dedup_key` | string | Canonical paper identifier (DOI > PMID > title hash) |
| `first_seen_run` | string | Run ID when first encountered |
| `last_seen_run` | string | Run ID when most recently seen |
| `source_ids` | list[string] | All source-specific IDs across runs |

### paper_status.parquet
| Column | Type | Description |
|---|---|---|
| `dedup_key` | string | Canonical paper identifier |
| `status` | string | `active`, `suppressed`, `starred`, `dismissed` |
| `suppressed_reason` | string | Why suppressed (if applicable) |
| `updated_at` | datetime | Last status change timestamp |

### run_index.parquet
| Column | Type | Description |
|---|---|---|
| `run_id` | string | Unique run identifier (ISO timestamp) |
| `timestamp` | datetime | Run start time (UTC) |
| `mode` | string | `daily`, `dry_run`, `backfill`, `manual` |
| `paper_count` | int | Total papers after filtering |
| `health_passed` | bool | Whether all health gates passed |
| `diagnostics` | struct | Source counts, timing, warnings |

### llm_cache.parquet
| Column | Type | Description |
|---|---|---|
| `prompt_hash` | string | SHA-256 of the full prompt |
| `response` | string | JSON-serialized LLM response |
| `model` | string | Model identifier |
| `created_at` | datetime | Cache entry timestamp |
| `tokens_used` | int | Total tokens for this request |

## Data Flow Between Phases

```
RawRecord[]  ──normalize──►  CanonicalPaper[]  ──dedup──►  CanonicalPaper[]
  (per source)                   (merged)                    (unique)

CanonicalPaper[]  ──enrich──►  CanonicalPaper[]  ──filter──►  CanonicalPaper[]
  (unique)                      (+TLDR, +citations)           (quality-gated)

CanonicalPaper[]  ──score──►  ScoreResult[]  ──triage──►  LlmTriageResult[]
  (quality-gated)               (11 signals)                 (adjusted)

ScoreResult[] + LlmTriageResult[] + ClinicalTrialAnnotation[]
  ──render──►  review_data.json → report.md, papers.csv, zotero.csv, ...
  ──state───►  seen_papers.parquet, paper_status.parquet, run_index.parquet
```
