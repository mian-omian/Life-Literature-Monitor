# Workflow Contract

This file defines operating workflow semantics for a rebuild.

## Local Workflow

Safe inspection:

```powershell
uv run lifelit --help
uv run lifelit validate-config --config config
uv run lifelit retrieval simulate --config config
uv run lifelit readiness --config config
uv run lifelit review --dry-run
uv run lifelit notify --dry-run
```

Artifact-writing or external-operation commands:

```powershell
uv run lifelit run --config config --mode dry_run
uv run lifelit run --config config
uv run lifelit review
uv run lifelit notify
uv run lifelit config-promote
```

## Health and Promotion

A run writes a timestamped snapshot first. It promotes to `outputs/latest/`
only when health gates pass. Degraded runs must preserve diagnostics and must
not be described as clean successful reading surfaces.

Minimum gates:

- source health;
- output health;
- curation size;
- exclude rate;
- provider-specific enrichment health where implemented;
- runtime partial-failure policy.

## GitHub Actions Expectations

A rebuilt repository may add:

- Validate workflow: dependency sync, config validation, Ruff, mypy, pytest.
- Daily workflow: readiness, config validation, run, staged state/output commit
  according to storage policy, notification, diagnostics.

Default validation workflows must not require real literature APIs, real LLM
calls, or SMTP sends.

## Review-Gated Config Evolution

Review UI writes feedback events. Feedback can produce proposals. Proposals can
produce config drafts. Config drafts require validation/diff/review before
promotion. No UI or feedback command silently mutates live config.
