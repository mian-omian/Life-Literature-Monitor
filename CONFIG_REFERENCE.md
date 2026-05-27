# Config Reference

Every configuration field across all 13 YAML files, with implementation status.

Authority note: this file is supporting documentation. For current operating
guidance, use `README.md`, `docs/user_guide.en.md`, `DESIGN.md`, and
`docs/config_interface_implementation_plan.md`. Current code registers 10
scoring signals; `penalty_signal` is not a registered signal.

**Status legend:**
- ✅ **Wired** — Field is read at runtime and affects behavior
- ⚠️ **Placeholder** — Field is validated by Pydantic but has no runtime effect
- 🔧 **Hardcoded** — Config value exists but behavior uses hardcoded defaults

---

## sources.yml

Data source enablement, roles, and API key mappings.

### `sources.pubmed`

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `enabled` | bool | `true` | ✅ | Gates PubMed retrieval in cli.py |
| `roles` | list[str] | `["formal_retrieval"]` | ⚠️ | Decorative — role is hardcoded in retrieval_strategy.py |
| `api_key_env` | str | `"NCBI_API_KEY"` | ✅ | Env var name for API key |
| `daily_date_type` | str | `"edat"` | ✅ | PubMed date field for daily queries |
| `backfill_date_type` | str | `"pdat"` | ⚠️ | Defined but never read; backfill logic is not implemented |

### `sources.europe_pmc`

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `enabled` | bool | `true` | ✅ | Gates Europe PMC retrieval |
| `roles` | list[str] | `["supplementary_formal_retrieval", "enrichment"]` | ⚠️ | Decorative |

### `sources.biorxiv`

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `enabled` | bool | `true` | ✅ | Gates bioRxiv retrieval |
| `roles` | list[str] | `["preprint_retrieval"]` | ⚠️ | Decorative |
| `date_field` | str | `"posted_date"` | ⚠️ | Accepted by retriever but docstring says "Ignored (present for caller consistency)" |

### `sources.semantic_scholar`

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `enabled` | bool | `true` | ✅ | Gates Semantic Scholar retrieval |
| `roles` | list[str] | `["recommendation_retrieval", "standard_enrichment", "deep_enrichment"]` | ⚠️ | Decorative |
| `api_key_env` | str | `"SEMANTIC_SCHOLAR_API_KEY"` | ✅ | Env var name for API key |

### `sources.openalex`

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `enabled` | bool | `true` | ✅ | Gates OpenAlex enrichment |
| `roles` | list[str] | `["enrichment"]` | ⚠️ | Decorative |
| `api_key_env` | str | `"OPENALEX_API_KEY"` | ✅ | Passed to enrichment via `enrich_all(openalex_api_key_env=...)` |

### `sources.crossref`

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `enabled` | bool | `true` | ✅ | Gates Crossref enrichment |
| `roles` | list[str] | `["enrichment"]` | ⚠️ | Decorative |
| `mailto_env` | str | `"CROSSREF_MAILTO"` | ✅ | Passed to enrichment via `enrich_all(crossref_mailto_env=...)` |
| `user_agent_env` | str | `"CROSSREF_USER_AGENT"` | ✅ | Passed to enrichment via `enrich_all(crossref_user_agent_env=...)` |

---

## runtime.yml

Pipeline execution parameters.

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `mode` | str | `"daily"` | ✅ | Run mode; CLI can override |
| `timezone` | str | `"Asia/Shanghai"` | ⚠️ | Defined but never applied — pipeline hardcodes UTC |
| `newest_offset_days` | int | `1` | ✅ | Exclude papers newer than N days |
| `oldest_offset_days` | int | `7` | ✅ | Look back N days |
| `partial_failure_policy` | str | `"warn"` | ⚠️ | Defined but never checked at runtime |

---

## output.yml

Output format toggles. Each maps to a render function in `render.py`.

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `review_data_json` | bool | `true` | ✅ | Structured JSON for downstream consumers |
| `report_md` | bool | `true` | ✅ | Human-readable Markdown briefing |
| `papers_csv` | bool | `true` | ✅ | Flat CSV of all papers |
| `zotero_csv` | bool | `true` | ✅ | Zotero-importable CSV |
| `suppressed_csv` | bool | `true` | ✅ | CSV of filtered-out papers |
| `run_summary_json` | bool | `true` | ✅ | Run-level statistics |
| `static_html` | bool | `false` | ⚠️ | **No renderer exists.** Config placeholder only |

---

## filters.yml

Config-driven filter rules. **Schema exists but pipeline uses hardcoded rules.**

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `filters` | list[FilterRule] | `[]` | ⚠️ | FilterRule model is fully specified (targets, operators, actions) but never consumed. Pipeline uses 9 hardcoded rules in filtering.py |

---

## privacy.yml

Privacy controls. **Entire module is a placeholder — no field is read at
runtime.**

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `commit_llm_cache` | bool | `false` | ⚠️ | Never read; LLM cache is committed unconditionally |
| `commit_raw_llm_prompts` | bool | `false` | ⚠️ | Never read; raw prompts are never stored |
| `commit_raw_api_payloads` | bool | `false` | ⚠️ | Never read; raw payloads are never stored |
| `include_abstracts_in_outputs` | bool | `true` | ⚠️ | Never read; abstracts are always included in outputs |

---

## llm.yml

LLM provider, eligibility, budget, and cache configuration. **All fields are
fully wired.**

### `provider`

| Field | Type | Default | Status |
|---|---|---|---|
| `base_url_env` | str | (env var name) | ✅ |
| `api_key_env` | str | (env var name) | ✅ |
| `model_env` | str | (env var name) | ✅ |

### Top-level

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `eligible_sections` | list[str] | `["semantic_recommendations", "published", "preprinted"]` | ✅ | Controls which sections get LLM triage |
| `max_papers_per_run` | int | `0` | ✅ | Hard cap on triaged papers per run |
| `max_papers_per_section` | int | `0` | ✅ | Cap per section |
| `max_score_adjustment` | float | `0.0` | ✅ | LLM score adjustment clamp |

### `budget`

| Field | Type | Default | Status |
|---|---|---|---|
| `per_run_token_limit` | int | `200000` | ✅ |
| `per_paper_token_limit` | int | `4000` | ✅ |
| `per_run_cost_limit` | float | `2.00` | ✅ |
| `per_paper_cost_limit` | float | `0.05` | ✅ |
| `cost_per_1k_input_tokens` | float | `0.001` | ✅ |
| `cost_per_1k_output_tokens` | float | `0.002` | ✅ |

### `cache`

| Field | Type | Default | Status |
|---|---|---|---|
| `enabled` | bool | `true` | ✅ |
| `path` | str | `"state/llm_cache.parquet"` | ✅ |

### `fallback_tldr`

| Field | Type | Default | Status |
|---|---|---|---|
| `enabled` | bool | `true` | ✅ |
| `max_length` | int | `280` | ✅ |

---

## storage.yml

State and output directory paths, commit policy, and retention.

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `state_dir` | str | `"state"` | ✅ | Parquet file directory |
| `output_dir` | str | `"outputs/latest"` | ✅ | Promoted output directory |
| `feedback_events_dir` | str | `"feedback/events"` | ✅ | Feedback YAML directory |
| `commit_state` | bool | `true` | ✅ | Checked by readiness.py and CI workflow |
| `commit_outputs_latest` | bool | `true` | ✅ | Checked by readiness.py and CI workflow |
| `commit_output_snapshots` | bool | `true` | ⚠️ | Defined but never checked at runtime |
| `retain_runs` | int | `30` | ✅ | Snapshot retention count |
| `atomic_writes` | bool | `true` | ⚠️ | Defined but never checked; atomic writes are always used |

---

## scoring.yml

Scoring sections, registered signals, and per-section overrides.

### `sections`

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `sections` | list[str] | `["author_tracked_manual", "published", "preprinted", "semantic_recommendations"]` | ⚠️ | Never read from config; hardcoded in `PRESET_CONTEXTS` |

### `signals`

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `signals` | list[str] | 10 signal names | ⚠️ | Never read from config; hardcoded in `REGISTERED_SIGNALS` |

### `section_overrides`

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `section_overrides` | dict[str, SectionOverride] | `{published: {journal_signal: 2.0}, preprinted: {journal_signal: 2.0}}` | ⚠️ | **Entire model (~88 lines) is validated but never applied to scoring contexts.** The overrides in the config file have zero effect. |

### Signal Implementation Status

Each of the 10 signals is registered in `REGISTERED_SIGNALS` and has a
computer function in `scoring.py`:

| Signal | Status | Behavior |
|---|---|---|
| `entry_signal` | ✅ | Maps entry types to base scores (author_tracked=1.0 → semantic_recommendation=0.5) |
| `topic_signal` | ✅ | Per-topic tiered saturation (core: binary 0.50, context: K=2 × 0.25, support: K=5 × 0.15, broad: K=5 × 0.10); max across topics |
| `journal_signal` | ✅ | ISSN/name/alias matching against reputation tiers with calibrated raw signal |
| `recommendation_profile_signal` | ✅ | Extracts S2 profile weights from source records; returns best weight or 0.5 neutral |
| `recommendation_rank_signal` | ✅ | Linear decay from rank 1 to configured min_rank; returns 0.5 neutral when rank data absent |
| `recency_signal` | ✅ | Exponential decay with configurable half-life (published: 365d, preprint: 90d) |
| `metadata_quality_signal` | ✅ | Fraction of present metadata fields (7 fields: doi, pmid, abstract, title, journal, authors, date) |
| `access_signal` | ✅ | Open Access status from Europe PMC / OpenAlex enrichment |
| `preprint_status_signal` | ✅ | Rewards preprints with published versions (0.5 + bonus when published, 0.0 otherwise) |
| `state_signal` | ⚠️ | **Stub** — always returns 0.0 ("no prior state") |

There is no registered `penalty_signal` in current source. Penalties and
boundary effects are represented as score-result metadata, warnings, or
rule/scoring adjustments, not as a configurable registered signal.

---

## journals.yml

Journal whitelist with canonical names, reputation tiers, and metadata.

| Field | Type | Status | Notes |
|---|---|---|---|
| `name` | str | ✅ | Canonical journal name |
| `aliases` | list[str] | ✅ | Alternative names for normalization matching |
| `issn` | list[str] | ✅ | Print ISSN(s) for matching |
| `eissn` | list[str] | ✅ | Electronic ISSN(s) for matching |
| `roles` | list[str] | ✅ | `normalize`, `reputation`, `formal_retrieval` — consumed by normalization, scoring, and retrieval |
| `reputation_tier` | str \| null | ✅ | `top`, `high`, `standard`, or null — mapped to scores in `_effective_journal_signal()` |
| `journal_signal_coefficient` | float \| null | ✅ | Modulates high-tier journal scores to [0.5, 1.0] |
| `clinical_trial_high_level` | bool | ✅ | Wired through clinical.py → render.py |
| `openalex_source_id` | str \| null | ⚠️ | Ingestible by profile pipeline but **never used at runtime** |
| `nlm_abbreviation` | str \| null | ⚠️ | Same as openalex_source_id — profile-only dead end |

---

## topics.yml

Research topic definitions driving retrieval and scoring.

| Field | Type | Status | Notes |
|---|---|---|---|
| `name` | str | ✅ | Topic identifier |
| `id` | str | ✅ | Same as name |
| `label` | str | ✅ | Same as name |
| `retrieval_logic` | str | ⚠️ | `MESH_OR_KEYWORDS` only; `MESH_ONLY` and `KEYWORDS_ONLY` have identical behavior |
| `priority` | str | ✅ | `strong` vs `secondary` affects entry signal |
| `intent` | str | ✅ | Human-readable; not used in retrieval/scoring |
| `keywords` | list[str] | ✅ | Legacy flat keyword list |
| `mesh_terms` | list[str] | ✅ | MeSH terms for retrieval |
| `pubmed_mesh_terms` | list[str] | ✅ | PubMed-specific MeSH clauses |
| `pubmed_free_text_roots` | list[str] | ✅ | PubMed free-text search roots |
| `europe_pmc_title_abstract_roots` | list[str] | ✅ | Europe PMC TITLE/ABSTRACT search |
| `biorxiv_filter_terms` | list[str] | ✅ | Post-retrieval bioRxiv filter |
| `scoring_terms` | list[str] | ✅ | Flat scoring terms (legacy mode) |
| `llm_context_terms` | list[str] | ✅ | Injected into LLM prompt context |
| `core_terms` | list[str] | ✅ | Tier 1 scoring terms |
| `context_terms` | list[str] | ✅ | Tier 2 scoring terms |
| `broad_terms` | list[str] | ✅ | Tier 3 scoring terms |
| `support_terms` | list[str] | ✅ | Tier 4 scoring terms |
| `boundary_terms` | list[str] | ✅ | Tier 5 scoring terms (not in current config) |

---

## seeds.yml

Seed papers for Semantic Scholar recommendations.

| Field | Type | Status | Notes |
|---|---|---|---|
| `positive_seeds` | list[str] | ✅ | DOI list → S2 recommendations API |
| `negative_seeds` | list[str] | ✅ | Passed to S2 API as exclusion signal |
| `boundary_seeds` | list[str] | ⚠️ | Compiled into strategy but **never passed to the S2 API call** |
| `seed_profiles` | list[SeedProfile] | ⚠️ | **Entire SeedProfile model is dead code** — never read by any source file |

---

## semantic_scholar.yml

Semantic Scholar API parameters.

| Field | Type | Default | Status | Notes |
|---|---|---|---|---|
| `recommendation_profiles` | list[RecommendationProfile] | `[]` | ⚠️ | **Entire RecommendationProfile model is dead code** |
| `recommendation_limit` | int | `20` | ✅ | Passed to S2 API |
| `standard_enrichment_fields` | list[str] | `["tldr"]` | ✅ | Passed to enrichment path via `s2_fields` |
| `deep_enrichment_top_n` | int | `0` | ✅ | Read by CLI orchestrator; gates deep S2 enrichment step |
| `search_enabled_for_daily` | bool | `false` | ⚠️ | Defined but never read |

---

## authors.yml

Tracked author ORCID IDs.

| Field | Type | Status | Notes |
|---|---|---|---|
| `author_id` | str | ✅ | ORCID identifier |
| `author_id_type` | str | ✅ | Always `"orcid"` |

All 54 author entries in the current config are fully functional.

---

## Summary: Implementation Coverage by Config File

| Config File | Total Fields | Wired | Placeholder | Coverage |
|---|---|---|---|---|
| `sources.yml` | 20 | 15 | 5 | 75% |
| `runtime.yml` | 5 | 3 | 2 | 60% |
| `output.yml` | 7 | 6 | 1 | 86% |
| `filters.yml` | 1 (list) | 0 | 1 | 0% |
| `privacy.yml` | 4 | 0 | 4 | 0% |
| `llm.yml` | 16 | 16 | 0 | 100% |
| `storage.yml` | 8 | 6 | 2 | 75% |
| `scoring.yml` | 3 (top-level) + 10 signals | 0 config fields, 9/10 signals | top-level config surface + 1 neutral/default signal | 0% config, 90% signals |
| `journals.yml` | 11 | 9 | 2 | 82% |
| `topics.yml` | 18 | 17 | 1 | 94% |
| `seeds.yml` | 4 | 2 | 2 | 50% |
| `semantic_scholar.yml` | 5 | 3 | 2 | 60% |
| `authors.yml` | 2 | 2 | 0 | 100% |
| **Total** | **104** | **79** | **25** | **~76%** |

Note: this table is approximate supporting documentation, not executable
evidence. "Wired" counts should be revalidated by a future config-to-runtime
contract task before being used as a completion claim.
