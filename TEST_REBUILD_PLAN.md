# Test Rebuild Plan

This file defines the minimum test suite shape for a rebuild.

## Test Rules

- Default tests must be offline and fixture/mocked.
- Real API tests must be marked integration and deselected by default.
- Tests must use repository-local pytest temporary storage.
- Tests must cover both happy paths and degraded/partial-failure paths.

## Required Test Areas

| Area | Required coverage |
|---|---|
| Config | 13-file load, strict unknown-field rejection, secret scanning, cross-file validation |
| Models | minimum identifiability, schema validation, serialized shapes |
| Retrieval strategy | source roles, query compilation, date windows, warnings/hard errors |
| Source adapters | mocked success/failure/rate-limit/partial-failure scenarios |
| Normalization | source-specific fields to CandidatePaper |
| Dedup | DOI/PMID/PMCID/S2/OpenAlex/preprint/title fallback behavior |
| Enrichment | mocked provider enrichment, warnings, retry behavior, no duplicate warning explosion |
| Filtering | rule order, configured filters, formal journal whitelist, bypasses, hard suppressions |
| Scoring | 10 signals, section routing, section overrides, ranking/banding |
| LLM | budget, cache, strict JSON, repair path, per-paper failure, no retrieval/config mutation |
| Render | JSON, Markdown, CSV, Zotero, suppressed CSV, run summary |
| State | Parquet read/write, atomic write diagnostics, suppression identifiers |
| Feedback/proposals | events, apply/list/undo, proposal and config-draft gates |
| CLI | documented commands match Click registry, command options, dry-run/write boundaries |
| Readiness | offline checks, effective-config summaries, warn/not-ready behavior |
| Notification | dry-run preview, degraded/failed subject behavior, secret redaction |

## Full Validation

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

## Documentation Drift Tests

The rebuilt repository should include lightweight tests that verify:

- documented CLI top-level commands match the Click registry;
- documented scoring signals match the registry;
- static HTML and Semantic Scholar daily Search are deferred unless
  implemented;
- docs do not describe active config surfaces as dead or unwired.
