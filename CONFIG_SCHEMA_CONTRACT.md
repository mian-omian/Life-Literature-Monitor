# Config Schema Contract

LifeLit uses exactly 13 YAML config files under `config/`.

## Files

```text
authors.yml
filters.yml
journals.yml
llm.yml
output.yml
privacy.yml
runtime.yml
scoring.yml
seeds.yml
semantic_scholar.yml
sources.yml
storage.yml
topics.yml
```

Unknown fields must be rejected unless a milestone explicitly adds them.

## Required Config Areas

| File | Required behavior |
|---|---|
| `authors.yml` | Stable-ID tracked author entries; no runtime pure-name tracking |
| `filters.yml` | Typed enabled/disabled filter rules consumed by filtering |
| `journals.yml` | Names, aliases, ISSNs/eISSNs, roles, reputation tiers, raw signal/coefficient fields |
| `llm.yml` | Provider env names, eligibility, caps, budget, cache, fallback TLDR |
| `output.yml` | Toggles for JSON, Markdown, CSV, Zotero, suppressed CSV, run summary, planned static HTML |
| `privacy.yml` | Abstract redaction and raw-artifact policy visibility |
| `runtime.yml` | Mode, timezone, offsets, partial-failure policy, novelty/S2 suppression settings |
| `scoring.yml` | Sections, registered signals, section overrides |
| `seeds.yml` | Global seeds, boundary seeds, seed profiles |
| `semantic_scholar.yml` | Recommendation profiles, limits, enrichment fields, deep enrichment top N, deferred search |
| `sources.yml` | Enabled flags, roles, API/contact env names |
| `storage.yml` | State/output/feedback dirs, commit policy, retention, atomic writes |
| `topics.yml` | Retrieval terms, scoring terms, tiered topic terms, LLM context terms |

## Secret Rule

Config may contain environment variable names. It must not contain real API
keys, SMTP passwords, tokens, `.env` values, or raw credentials.

## Validation Command

```powershell
uv run lifelit validate-config --config config
```

The command must validate all 13 files, reject schema errors, reject embedded
secrets, and report success/failure clearly.
