# Config Reference

This is a supporting, source-current overview of the 13 YAML config files.
`src/lifelit/config.py`, `DESIGN.md`, and `docs/user_guide.en.md` remain
authoritative.

Status legend:

- **Active** — read by the current run path and affects behavior or visible
  effective-config reporting.
- **Evidence-only** — compiled or shown as evidence but not a full behavior
  control.
- **Planned** — accepted or documented for future work, with no current runtime
  implementation.

## sources.yml

Source enablement, source roles, and environment-variable names are active.
The run path uses roles to decide formal retrieval, supplementary retrieval,
preprint retrieval, recommendation retrieval, standard/deep enrichment, and
provider-specific env forwarding.

Key notes:

- PubMed and Europe PMC formal retrieval use compiled topic/journal strategy.
- bioRxiv uses posted-date style retrieval and local topic filtering.
- Semantic Scholar Recommendations are active; daily Search is deferred.
- OpenAlex and Crossref enrichment use configured contact/API env names.

## runtime.yml

Active fields:

- `mode`
- `timezone`
- `newest_offset_days`
- `oldest_offset_days`
- `partial_failure_policy`
- `skip_novelty_filter`
- `s2_suppression_cooldown_days`

Runtime timezone participates in visible run-window/report policy. Partial
failure policy is applied by runtime health handling.

## output.yml

Active outputs:

- `review_data_json`
- `report_md`
- `papers_csv`
- `zotero_csv`
- `suppressed_csv`
- `run_summary_json`

Planned output:

- `static_html` is planned only. No static HTML renderer exists in the current
  code.

## filters.yml

Configured filter rules are active. The run path forwards `config.filters` into
`apply_filters`, where enabled rules are evaluated in YAML order after the
technical/bypass/persisted-suppression/formal-journal-whitelist guardrails and
before later hardcoded publication/recommendation/preprint/metadata/quality
guardrails.

## privacy.yml

Privacy policy is active for output abstract redaction and secret-safe runtime
visibility. Raw prompt and raw API payload policy is represented as
`no_new_raw_writes`; raw secret values must never be committed.

## llm.yml

LLM provider env names, eligible sections, paper caps, score adjustment limits,
budget, cache, and fallback TLDR settings are active when LLM triage is enabled
and the required environment variables are present.

## storage.yml

Storage policy is active for state/output directories, atomic writes, latest
promotion, snapshot retention, and workflow staging policy. Commit flags govern
what the workflow stages; they do not grant agents permission to commit or push.

## scoring.yml

Scoring uses 10 registered signals. Section overrides are active in the run
path and merge into runtime scoring contexts after preset, topic, and journal
context construction.

The four macro sections are:

- `author_tracked_manual`
- `published`
- `preprinted`
- `semantic_recommendations`

Registered signals:

- `entry_signal`
- `topic_signal`
- `journal_signal`
- `recommendation_profile_signal`
- `recommendation_rank_signal`
- `recency_signal`
- `metadata_quality_signal`
- `access_signal`
- `preprint_status_signal`
- `state_signal`

## journals.yml

Journal entries are active for:

- canonical name/alias/ISSN/eISSN matching;
- normalization;
- formal published whitelist filtering;
- reputation scoring;
- clinical-trial venue annotation.

`openalex_source_id` and `nlm_abbreviation` are accepted identifiers and may be
used by profile/config workflows and identity evidence; current runtime matching
primarily relies on name, alias, ISSN, and eISSN.

## topics.yml

Topic fields are active across retrieval, scoring, and LLM context according to
their specific field:

- PubMed and Europe PMC fields drive formal query clauses.
- bioRxiv filter terms drive local preprint filtering.
- tiered topic terms drive per-topic saturation scoring.
- LLM context terms are injected into triage prompts.

Topic tier scoring is scoring behavior only; it does not silently rewrite
retrieval semantics.

## seeds.yml

Global seeds and seed profiles are active for Semantic Scholar recommendation
retrieval where enabled. Boundary seeds are evidence-only calibration material;
they are not a complete behavior-driving negative or suppression mechanism.

## semantic_scholar.yml

Active fields:

- `recommendation_profiles`
- `recommendation_limit`
- `standard_enrichment_fields`
- `deep_enrichment_top_n`

Planned/deferred field:

- `search_enabled_for_daily` remains deferred; daily Semantic Scholar Search is
  not the current retrieval path.

## authors.yml

Tracked authors are active when configured with stable supported identifiers.
Runtime pure-name author tracking is outside the MVP boundary.
