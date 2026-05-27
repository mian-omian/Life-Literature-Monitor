# User Guide

## Prerequisites

- Python 3.12 or later
- [uv](https://docs.astral.sh/uv/) package manager
- A GitHub account (for scheduled runs via Actions)
- API keys for the sources you want to enable (see below)

## Quick Start

### 1. Clone and Install

```bash
git clone <your-private-repo-url> lifelit
cd lifelit
uv sync
```

### 2. Get API Keys

LifeLit needs API keys for the sources you enable. All keys go into GitHub
Actions secrets (or a local `.env` file for development).

| Source | Key | Registration Link |
|---|---|---|
| PubMed | `NCBI_API_KEY` | https://account.ncbi.nlm.nih.gov/settings/ |
| Semantic Scholar | `SEMANTIC_SCHOLAR_API_KEY` | https://api.semanticscholar.org/api-docs/ |
| OpenAlex | `OPENALEX_API_KEY` | https://openalex.org/account |
| Crossref | `CROSSREF_MAILTO` | https://crossref.org/ (polite pool email) |
| LLM Provider | `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL` | Your OpenAI-compatible provider |

Email notification (optional):

| Setting | Purpose |
|---|---|
| `EMAIL_SMTP_HOST` | SMTP server address |
| `EMAIL_SMTP_PORT` | SMTP port (typically 587) |
| `EMAIL_SMTP_USERNAME` | SMTP login |
| `EMAIL_SMTP_PASSWORD` | SMTP password |
| `EMAIL_TO` | Recipient address |
| `EMAIL_FROM` | Sender address |

### 3. Configure Your Research Topics

Edit `config/topics.yml` to define your research interests. Each topic has:

```yaml
topics:
  - name: core-my-research-area
    id: core-my-research-area
    label: core-my-research-area
    retrieval_logic: MESH_OR_KEYWORDS
    priority: strong
    intent: "Human-readable description of this topic"
    keywords:
      - "keyword one"
      - "keyword two"
    mesh_terms:
      - "MeSH Term One"
      - "MeSH Term Two"
    pubmed_mesh_terms:
      - "MeSH Term One[MeSH Terms]"
    pubmed_free_text_roots:
      - "keyword root"
    europe_pmc_title_abstract_roots:
      - "search term"
    biorxiv_filter_terms:
      - "filter keyword"
    scoring_terms:
      - "scoring keyword"
    llm_context_terms:
      - "context for LLM"
    core_terms:
      - "highest priority term"
    context_terms:
      - "medium priority term"
    broad_terms:
      - "broad interest term"
    support_terms:
      - "supporting term"
```

**Priority levels:**
- `strong` — core research area; papers get higher entry scores
- `secondary` — adjacent interest; papers get lower entry scores

**Tiered scoring terms** (`core_terms`, `context_terms`, `broad_terms`,
`support_terms`, `boundary_terms`) enable fine-grained topic signal calculation.
If none are provided, the scorer falls back to flat `scoring_terms`.

### 4. Configure Journals

Edit `config/journals.yml` to build your journal whitelist. Each entry:

```yaml
journals:
  - name: "Nature"
    aliases: []
    issn: ["0028-0836"]
    eissn: ["1476-4687"]
    roles: ["normalize", "reputation"]
    reputation_tier: "top"
    openalex_source_id: null
    nlm_abbreviation: null
    clinical_trial_high_level: false
    journal_signal_coefficient: 1.428571
```

**Reputation tiers:** `top`, `high`, `standard`, `null` (unranked/watch).

**Roles:**
- `normalize` — use this entry for journal name normalization (default)
- `reputation` — use this entry's tier for scoring
- `formal_retrieval` — restrict PubMed/Europe PMC retrieval to this journal

**Clinical trial detection:** Set `clinical_trial_high_level: true` for
journals where you want papers flagged as potential clinical trials.

### 5. Add Seed Papers

Edit `config/seeds.yml` to add DOIs of papers that exemplify your interests.
These drive Semantic Scholar's recommendation engine:

```yaml
positive_seeds:
  - "10.1126/science.adm7969"
  - "10.1038/s41586-024-07623-6"
```

### 6. Track Authors

Edit `config/authors.yml` to add ORCID IDs of researchers whose new papers you
always want to see:

```yaml
authors:
  - author_id: "0000-0002-1825-0097"
    author_id_type: "orcid"
```

### 7. Validate Configuration

```bash
uv run lifelit validate-config --config config
```

This checks all 13 config files for schema violations and cross-reference
integrity. Fix any errors before proceeding.

### 8. Dry Run

```bash
uv run lifelit run --config config --mode dry_run
```

A dry run skips the retrieval phase (no API calls) but still runs scoring and
rendering. To check what your config will retrieve before running, use:

```bash
uv run lifelit retrieval simulate --config config
```

This compiles the retrieval strategy and reports query counts, source roles,
selected journals, filter terms, and any warnings or hard errors — all offline.

### 9. Full Local Run

```bash
uv run lifelit run --config config
```

This executes the full pipeline. Output appears in `outputs/<run_id>/`. If
health gates pass, it's promoted to `outputs/latest/`.

### 10. Review Papers

```bash
uv run lifelit review
```

Launches a Streamlit UI at `http://localhost:8501` for browsing, searching,
starring, and suppressing papers from the latest run.

### 11. Set Up GitHub Actions

1. Push your configured repository to GitHub (private repository recommended)
2. Add all API keys as [repository secrets](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
3. The daily workflow runs automatically at ~14:07 CST each day
4. Check `outputs/latest/report.md` for your daily briefing

---

## Daily Usage

Once set up, LifeLit runs automatically. Your daily workflow:

1. **Receive email** notification with paper counts and highlights
2. **Read `outputs/latest/report.md`** for the full briefing
3. **Optionally run `lifelit review`** to interactively browse and curate papers
4. **Periodically refine** your topics, journals, and seeds as your interests evolve

## Iterating on Configuration

LifeLit has a built-in feedback loop for continuous refinement:

### Feedback Events

Use the Review UI to star, suppress, or dismiss papers. These actions are saved
as YAML feedback events in `feedback/events/`.

```bash
uv run lifelit feedback apply   # Apply feedback to current state
uv run lifelit feedback list    # List pending feedback events
uv run lifelit feedback undo    # Undo a feedback event
```

### Config Proposals

Based on feedback patterns, LifeLit can suggest config changes:

```bash
uv run lifelit feedback propose     # Generate config proposals
uv run lifelit feedback config-draft  # Create a config draft from proposals
```

### Profile-Based Setup

Instead of editing YAML directly, you can write a natural-language profile:

```bash
uv run lifelit profile compile   # Compile profile text → config draft
uv run lifelit profile validate  # Validate a profile-generated draft
uv run lifelit profile diff      # Show what the draft would change
```

### Promoting Changes

After reviewing a config draft, promote it to live config:

```bash
uv run lifelit config-promote
```

Always run `validate-config` and a dry run after promotion.

---

## Architecture of Your Configuration

```
profile/                    config/                     state/
(NL descriptions)           (YAML declarations)         (Parquet data)
      │                          │                          │
      ▼                          ▼                          ▼
 profile compile ──► config_draft/ ──► config-promote ──► live config
                                                               │
 feedback events ──► proposals ──► config_draft/ ─────────────┘
```

This means you can start with a natural-language profile, compile it into a
config draft, refine it with feedback from actual runs, and promote it to live
config — all without writing YAML by hand.

---

## Troubleshooting

### "No papers retrieved"

- Check that your date window (`runtime.yml`) isn't too narrow
- Verify API keys are set and valid
- Run with `--verbose` to see per-source query details
- Check `outputs/<run_id>/run_summary.json` for source-level diagnostics

### "Config validation fails"

- Run `uv run lifelit validate-config --config config` for detailed errors
- Check that scoring signal names in section overrides exist in the registered signals list
- Ensure journal entries have valid `reputation_tier` values

### "GitHub Actions run fails"

- Check the Actions run log for step-level errors
- Verify all required secrets are set in repository settings
- Run locally first to isolate config vs. CI issues
- Check that `readiness` checks pass: `uv run lifelit readiness --config config`

### "LLM triage never runs"

- Verify `eligible_sections` in `llm.yml` is non-empty
- Check that `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL` are set
- Verify `budget.per_run_token_limit` and `budget.per_run_cost_limit` are not
  zero
- Check `state/llm_cache.parquet` — papers may be cached from prior runs
