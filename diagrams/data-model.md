# Data Model

```mermaid
classDiagram
    class RawRecord {
        +source
        +entry_type
        +retrieval_mode
        +query_id
        +profile_id
        +author_id
        +retrieved_time
        +source_record_id
        +source_url
        +raw_payload
    }

    class CandidatePaper {
        +doi
        +pmid
        +pmcid
        +semantic_scholar_paper_id
        +openalex_work_id
        +preprint_doi
        +title
        +abstract
        +authors
        +journal_name
        +journal_issn
        +journal_eissn
        +published_date
        +source
        +entry_type
        +source_record_id
    }

    class CanonicalPaper {
        +dedup_key
        +doi
        +pmid
        +pmcid
        +semantic_scholar_paper_id
        +openalex_work_id
        +preprint_doi
        +title
        +abstract
        +authors
        +journal_name
        +journal_issn
        +journal_eissn
        +published_date
        +sources
        +entry_types
        +source_records
        +tldr
        +tldr_source
        +openalex_enrichment
        +crossref_enrichment
        +europe_pmc_enrichment
        +publication_status
        +enrichment_warnings
    }

    class FilterDecision {
        +canonical_dedup_key
        +decision
        +reason
        +triggered_rules
    }

    class SectionScoreResult {
        +canonical_dedup_key
        +section
        +total_score
        +score_breakdown
        +matched_features
        +penalties
        +warnings
        +rank
        +band
        +explanation
        +deep_enrichment_eligible
        +llm_triage_selected
    }

    class LlmTriageResult {
        +canonical_dedup_key
        +section
        +selected_for_triage
        +triage_reason
        +llm_score_adjustment
        +llm_raw_response
        +cache_hit
        +llm_model
        +llm_tokens_used
        +llm_cost_estimate
        +fallback_tldr
        +warnings
    }

    class ReviewData {
        +run_id
        +generated_at
        +sections
        +total_papers
        +section_counts
    }

    RawRecord --> CandidatePaper : normalize
    CandidatePaper --> CanonicalPaper : deduplicate
    CanonicalPaper --> FilterDecision : filter
    CanonicalPaper --> SectionScoreResult : score
    SectionScoreResult --> LlmTriageResult : triage selection
    CanonicalPaper --> ReviewData : render
    SectionScoreResult --> ReviewData : render
```

## State Files

Important state files include:

| File | Purpose |
|---|---|
| `state/seen_papers.parquet` | Seen-paper and first/last run tracking |
| `state/paper_status.parquet` | Review status and suppression state |
| `state/run_index.parquet` | Run history and health metadata |
| `state/llm_cache.parquet` | LLM response cache when enabled |
| `state/proposals/` | Review/config proposal files |
| `state/config_draft/` | Reviewable config drafts |
