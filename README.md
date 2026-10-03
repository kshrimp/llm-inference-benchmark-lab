# LLM inference benchmark lab

A bounded serving case study for measuring LLM inference behavior with SGLang on Modal. The project focuses on a trustworthy baseline, one concurrency tradeoff, one lifecycle or failure investigation, and a short customer-facing recommendation based on recorded evidence.

## Scope

- One small dense text model, one GPU type, one precision, and one worker at a time.
- SGLang is the primary serving engine.
- Measurements distinguish offline throughput, client-observed serving latency, and any supported in-engine scheduler metrics.
- Missing metrics are recorded as null with reasons instead of inferred or fabricated.
- Container cold starts, model-file cache state, and prefix-cache state are tracked separately.

## Repository Layout

| Path | Purpose |
| --- | --- |
| `docs/PROJECT_REFERENCE.md` | Technical scope, methodology, data contracts, and acceptance criteria |
| `docs/ENVIRONMENT.md` | Local `uv` setup, package management, and Modal environment notes |
| `results/README.md` | Guidance for storing reproducible run artifacts |
| `get_started.py` | Small local environment check |

## Status

The project is in its initial setup phase. No publishable performance results are included in this repository yet.

## Development Setup

Use `uv` to recreate the local Python environment from `pyproject.toml` and `uv.lock`:

```bash
uv sync
uv run python get_started.py
```

See `docs/ENVIRONMENT.md` for package-management workflow and the distinction between the local venv and the remote Modal image.
