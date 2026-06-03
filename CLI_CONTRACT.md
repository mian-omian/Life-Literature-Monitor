# CLI Contract

The rebuilt CLI must expose these top-level commands:

```text
lifelit version
lifelit validate-config
lifelit run
lifelit review
lifelit notify
lifelit readiness
lifelit import-manual
lifelit config-promote
lifelit feedback ...
lifelit retrieval simulate
lifelit config migrate-real-use
lifelit profile ...
```

## Required Command Behavior

### `lifelit validate-config`

Required option:

- `--config, -c`

Behavior: validate 13 YAML files and secret safety. No network calls.

### `lifelit run`

Required option:

- `--config, -c`

Optional:

- `--mode`
- `--newest-offset-days`
- `--oldest-offset-days`

Behavior: execute the pipeline. `dry_run` skips source retrieval but can still
write local output/state artifacts.

### `lifelit retrieval simulate`

Behavior: compile strategy offline and report source roles, query counts,
selected journals, seeds/profiles, warnings, hard errors, and effective-config
summary.

### `lifelit readiness`

Behavior: offline readiness checks. No real API, LLM, or SMTP calls.

### `lifelit review`

Behavior: launch Streamlit unless `--dry-run` is provided. Dry-run prints the
launch command.

### `lifelit notify`

Behavior: render notification preview with `--dry-run`; SMTP send only without
dry-run and with configured env vars.

### Feedback and Profile Commands

Feedback commands must write events, applied-state output, proposals, or config
drafts. Profile commands must compile/validate/diff reviewable config drafts.
Neither path may silently mutate live config.

## CLI Drift Tests

The rebuild must include tests that compare documented top-level commands with
the Click command registry.
