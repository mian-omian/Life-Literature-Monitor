# Architecture

LifeLit is a single-process, batch-oriented Python application. The `lifelit`
Click CLI orchestrates a full run from strict YAML config to output artifacts
and state updates. It is designed for private local use and private GitHub
Actions operation.

This portal file is supporting documentation. In the full implementation
repository, `DESIGN.md`, `docs/user_guide.en.md`, and the source under
`src/lifelit/` remain the current authority. In the standalone portal
repository, those files are upstream implementation-repository references, not
local files.

## Module Map

```text
src/lifelit/
  cli.py                  Click commands and run orchestration
  config.py               Pydantic v2 config models for 13 YAML files
  models.py               RawRecord, CandidatePaper, CanonicalPaper, scores,
                           feedback, run metadata
  retrieval_strategy.py   Offline strategy compiler and simulator data
  pubmed.py               PubMed retrieval adapter
  europe_pmc.py           Europe PMC retrieval adapter
  biorxiv.py              bioRxiv retrieval adapter
  semantic_scholar.py     Recommendations, date filtering, and S2 helpers
  author_tracked.py       Stable-ID tracked-author retrieval
  retrieval.py            Manual import helpers
  normalization.py        RawRecord -> CandidatePaper normalization
  dedup.py                CandidatePaper -> CanonicalPaper deduplication
  enrichment.py           S2, OpenAlex, Crossref, Europe PMC, bioRxiv enrichment
  journal_matching.py     Shared journal identity matching
  filtering.py            Hard guardrails plus configured filter rules
  scoring.py              10 registered scoring signals and section contexts
  clinical.py             Clinical-trial venue annotation
  llm_client.py           OpenAI-compatible client
  prompts.py              LLM prompt construction and JSON contract
  budget.py               Token/cost budget accounting
  cache.py                LLM cache storage
  fallback_tldr.py        Deterministic fallback TLDR helper
  llm_triage.py           Section-aware bounded LLM triage
  render.py               Review data, Markdown, CSV, Zotero, summary rendering
  review_ui.py            Streamlit local review UI
  notify.py               Dry-run and SMTP notification rendering/sending
  readiness.py            Offline readiness checks and effective-config summary
  run_health.py           Promotion and runtime failure-policy gates
  source_reliability.py   Structured provider failure diagnostics
  feedback.py             Feedback event loading/apply/list/undo
  proposals.py            Proposal and config-draft handling
  profile_agent.py        Deterministic profile text parsing
  profile_resolver.py     Offline/optional-online identifier resolution
  profile_compiler.py     Profile/config draft compiler
  effective_config.py     Effective-config summaries for readiness/simulation
  state/
    __init__.py           Parquet/YAML state helpers
    suppression.py        S2 seen/rejected suppression identifiers
```

## Core Design Boundaries

- Runtime consumes strict config only. Natural-language interpretation belongs
  in the profile/config compiler and review-gated drafts.
- Papers and authors are identifier-first where required.
- Scoring is section-first. Scores are not comparable across macro sections.
- The scoring registry currently contains 10 signals.
- LLM triage is bounded, cache-backed, section-aware, and cannot retrieve,
  mutate config, or override hard suppressions.
- Feedback writes events or proposals. Config promotion is explicit and
  review-gated.
- No secrets belong in repository files.

## Runtime Shape

`lifelit run --config config` loads the 13 YAML config files, validates them,
computes the runtime date window, then proceeds through retrieval, processing,
rendering, state, health, and notification stages. The CLI command owns the
stage sequencing; individual modules provide deterministic helpers or source
adapters.

## Config and Effective Status

Several config surfaces are fully behavior-driving today, including configured
filter rules, source roles and environment-variable forwarding, Semantic
Scholar recommendation/seed profiles, date-window post-filtering, standard and
deep S2 enrichment controls, privacy output redaction, storage policy, runtime
timezone/partial-failure policy, and scoring section overrides.

Known deferred or incomplete surfaces remain explicit:

- `output.outputs.static_html` is planned only; no static HTML renderer exists.
- Semantic Scholar daily search is deferred; recommendations remain the active
  S2 daily retrieval mechanism.
- `boundary_seeds` are strategy evidence, not a complete calibration surface.

## Output and State

Runs write a timestamped output snapshot and may promote a healthy snapshot to
`outputs/latest/`. Primary artifacts include:

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

State is stored under `state/`, including seen papers, paper status, run index,
LLM cache, proposals, and config drafts where configured.
