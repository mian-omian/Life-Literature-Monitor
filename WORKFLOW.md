# Workflow

This document explains the current LifeLit run lifecycle and command safety
boundaries. It is supporting documentation. In the full implementation
repository, use `docs/user_guide.en.md` and the source for authoritative
details. In the standalone portal repository, those files are upstream
implementation-repository references, not local files.

## Command Risk Levels

### Read-only or dry inspection

These are the safest first commands:

```powershell
uv run lifelit --help
uv run lifelit validate-config --config config
uv run lifelit retrieval simulate --config config
uv run lifelit readiness --config config
uv run lifelit review --dry-run
uv run lifelit notify --dry-run
```

They validate or summarize local state without real retrieval, LLM calls, or
SMTP sends.

### Local artifact writers

These can write local files under configured directories:

```powershell
uv run lifelit run --config config --mode dry_run
uv run lifelit feedback apply
uv run lifelit feedback config-draft
uv run lifelit profile compile
uv run lifelit config-promote
```

`run --mode dry_run` skips source retrieval, but it is not read-only.

### External operations

These may call real services depending on config and environment variables:

```powershell
uv run lifelit run --config config
uv run lifelit notify
uv run lifelit profile compile --online
```

Treat these as owner-authorized operations only.

## Pipeline Lifecycle

```text
load config
-> validate config
-> compute runtime date window
-> compile retrieval strategy
-> retrieve enabled sources unless mode is dry_run
-> normalize RawRecord objects into CandidatePaper objects
-> deduplicate into CanonicalPaper objects
-> enrich metadata
-> apply filters
-> compute section-local scores
-> annotate clinical-trial venue signals
-> run bounded LLM triage for selected candidates
-> render full and curated review outputs
-> update seen/status/run-index state
-> run health gates
-> promote healthy output snapshot
-> write run summary and notify when configured
```

## Retrieval

The strategy compiler is the offline source of truth for query shape and source
roles. It compiles topics, journals, seeds, authors, and runtime date windows
into source-specific plans. Real retrieval adapters are called sequentially and
use fixed source-policy behavior for rate limits, retry discipline, and
secret-safe diagnostics.

Daily Semantic Scholar retrieval uses Recommendations, not the deferred daily
Search feature. Recommendations are locally post-filtered by the run date
window.

## Filtering

Filtering combines fixed guardrails with configured `filters.yml` rules.
Current order is:

1. technical exclude;
2. author-tracked bypass;
3. manual-import bypass;
4. persisted hard suppression;
5. formal published journal whitelist for `formal_retrieval` records;
6. configured filter rules in YAML order;
7. publication-type suppression;
8. recommendation guardrails;
9. preprint guardrails;
10. metadata completeness;
11. technical quality;
12. default keep.

Filtering is a guardrail layer, not relevance ranking.

## Scoring

The scorer registers 10 signals and computes weighted section-local totals.
The four macro sections are:

- `author_tracked_manual`
- `published`
- `preprinted`
- `semantic_recommendations`

`config/scoring.yml` section overrides are merged into runtime scoring contexts
after preset, topic, and journal context construction.

## LLM Triage

LLM triage is optional and bounded by:

- eligible sections;
- per-section and per-run paper caps;
- token and cost budget;
- cache;
- strict JSON/schema validation;
- bounded score adjustment.

LLM output can adjust section-local ranking, but it does not perform retrieval,
mutate config, or override hard suppressions.

## Outputs, State, and Promotion

The run path writes an audit review dataset, curated reading outputs, state
updates, and a run summary. Health gates decide whether a timestamped snapshot
is promoted to `outputs/latest/`. Degraded runs preserve diagnostics but should
not be described as clean successful reading surfaces.

## GitHub Actions

The Daily Pipeline can run the same CLI flow in GitHub Actions using repository
secrets. The Validate workflow runs local validation without real literature
APIs, real LLM calls, or SMTP sends.
