# LifeLit

**A personal life-science literature monitor that runs on a daily schedule.**

LifeLit retrieves newly indexed papers from multiple sources, normalizes and
deduplicates them, enriches with metadata, scores them with a multi-signal
engine, optionally runs LLM-based triage, and renders a reviewable daily
briefing — all from a private GitHub repository with zero infrastructure.

---

## What It Does

Every day, LifeLit:

1. **Fetches** new papers from PubMed, Europe PMC, bioRxiv, Semantic Scholar,
   and tracked-author profiles — scoped to your research topics.
2. **Normalizes** journal names, **deduplicates** across sources, and
   **enriches** with TLDRs, citations, and open-access indicators.
3. **Filters** out non-research content (editorials, retractions, incomplete
   metadata) through a 9-rule guardrail pipeline.
4. **Scores** every paper with 11 signals: topic relevance, journal reputation,
   recency, metadata quality, open access, preprint status, and more.
5. **Triages** top papers through an LLM (optional) with budget and cache
   controls to refine rankings.
6. **Renders** a reviewable report (Markdown, JSON, CSV, Zotero) and
   **notifies** you via email.
7. **Persists** state in Parquet files so every run knows what you've seen.

Output lands in `outputs/latest/`. No server, no database, no SaaS.

---

## Quick Navigation

| Document | For |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System design, module map, design rationale |
| [WORKFLOW.md](WORKFLOW.md) | Step-by-step pipeline walkthrough |
| [USER_GUIDE.md](USER_GUIDE.md) | Setup, configuration, daily usage |
| [CONFIG_REFERENCE.md](CONFIG_REFERENCE.md) | Every config field with implementation status |
| [AGENT_REPRODUCTION.md](AGENT_REPRODUCTION.md) | Step-by-step guide for coding agents to rebuild |
| [diagrams/pipeline-flow.md](diagrams/pipeline-flow.md) | Visual pipeline flowchart |
| [diagrams/data-model.md](diagrams/data-model.md) | Data model relationships |

---

## Architecture at a Glance

```
config/ (13 YAML files)
  │
  ▼
┌─────────────────────────────────────────────────────┐
│  Retrieval                                           │
│  PubMed · Europe PMC · bioRxiv · Semantic Scholar    │
│  Author Tracked                                      │
└───────────────────────┬─────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────┐
│  Processing Pipeline                                 │
│  Normalize → Dedup → Enrich → Filter → Score         │
│  (11-signal engine · journal reputation · topics)    │
└───────────────────────┬─────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────┐
│  LLM Triage (optional)                               │
│  Budget manager · Response cache · Fallback TLDR     │
└───────────────────────┬─────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────┐
│  Output & State                                      │
│  JSON · Markdown · CSV · Zotero · Email notify       │
│  State → Parquet (seen papers, status, run index)    │
└─────────────────────────────────────────────────────┘
```

---

## Technology

| Concern | Choice |
|---|---|
| Language | Python 3.12+ |
| Package manager | uv |
| CLI framework | Click |
| Config validation | Pydantic v2 |
| Review UI | Streamlit |
| State storage | PyArrow / Parquet |
| HTTP client | httpx |
| Type checking | mypy (strict mode) |
| Linting | ruff |
| CI/CD | GitHub Actions (daily cron + PR validate) |

---

## Philosophy

- **Local-first.** Everything runs in a GitHub Actions runner or your laptop.
  No cloud services, no backend, no SaaS lock-in.
- **Private by default.** Your research interests, reading history, and LLM
  prompts stay in your private repository.
- **Config-driven.** All behavior is declared in YAML. Change your topics,
  journals, authors, or scoring weights without touching code.
- **Snapshot-and-promote.** Each run writes to a timestamped directory. Only
  runs that pass health gates are promoted to `outputs/latest/`.
- **Agent-reproducible.** The architecture is documented at the level of detail
  needed for a coding agent to rebuild the project from scratch.
