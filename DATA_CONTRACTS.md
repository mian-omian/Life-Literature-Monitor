# Data Contracts

This file defines the important data objects and artifacts a rebuild must
preserve.

## Core Models

### RawRecord

Must include:

- source;
- entry type;
- retrieval mode;
- query/profile/author identifiers where applicable;
- retrieved timestamp;
- source record ID;
- source URL;
- raw payload.

### CandidatePaper

Normalized source-independent intermediate paper. Must have at least one stable
identifier or title plus source record ID.

### CanonicalPaper

Deduplicated and enriched paper. Must preserve identifiers, title/abstract,
authors, journal metadata, dates, sources, entry types, source records,
enrichment fields, dedup key, and warnings.

### FilterDecision

Must include canonical dedup key, decision, reason, and triggered rules.

Allowed decisions:

- `keep`
- `soft_suppress`
- `hard_suppress`
- `technical_exclude`

### SectionScoreResult

Must include canonical dedup key, section, total score, score breakdown,
matched features, penalties, warnings, rank, band, explanation, and downstream
deep-enrichment/LLM flags.

### LlmTriageResult

Must include canonical dedup key, section, selection state, triage reason,
bounded score adjustment, schema-validated raw response, cache status, model,
token/cost estimates, fallback TLDR, timestamp, and warnings.

## Output Artifacts

Required outputs:

```text
review_data_full.json
review_data.json
report.md
papers.csv
zotero.csv
suppressed.csv
run_summary.json
triage_results.json when LLM triage runs
```

`review_data.json` is the presentation source of truth. CSV exports are derived
from review data. Static HTML is deferred.

## State Artifacts

Required state areas:

```text
state/seen_papers.parquet
state/paper_status.parquet
state/run_index.parquet
state/llm_cache.parquet
state/proposals/
state/config_draft/
feedback/events/
```

State writes should be atomic where configured.

## Provenance Rule

No processing stage may discard source provenance needed to explain why a paper
appeared, how it was retrieved, and which source records contributed to it.
