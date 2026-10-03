# Environment and package management

This project uses `uv` for the local Python environment. The local environment is for development, validation scripts, the Modal CLI/client, and CPU checks. Modal GPU serving dependencies are pinned separately in the Modal image definition when the serving app is implemented.

## Files to commit

Commit these files so another checkout can recreate the same local environment:

| File | Purpose |
| --- | --- |
| `.python-version` | Records the intended Python version family for local development |
| `pyproject.toml` | Declares project metadata, Python requirement, and direct dependencies |
| `uv.lock` | Locks the full resolved dependency graph for reproducible installs |

Do not commit `.venv/`, `.env`, `.modal.toml`, caches, or generated result payloads.

## First-time setup

Install `uv` if it is not already available:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

From the repository root, create or update the virtual environment from the lockfile:

```bash
uv sync
```

This creates `.venv/` if needed and installs the locked dependencies.

## Activating the environment

You can activate the environment for an interactive shell:

```bash
source .venv/bin/activate
```

When active, the shell prompt usually starts with `(.venv)`. Confirm the interpreter with:

```bash
which python
python --version
```

You can also skip activation and run commands through `uv`:

```bash
uv run python get_started.py
uv run llm-inference-benchmark-lab
```

Using `uv run ...` is often clearer in documentation because it always uses the project environment.

## Adding packages

Add normal runtime/development packages with `uv add`:

```bash
uv add modal
uv add sglang
```

Add tools that are only needed for local development with `--dev`:

```bash
uv add --dev pytest ruff
```

After adding or removing packages, commit both `pyproject.toml` and `uv.lock`.

## Recreating the environment on another machine

Clone the repository, install `uv`, then run:

```bash
uv sync --locked
uv run python get_started.py
```

`--locked` tells `uv` to use the committed lockfile without changing it. If this fails, update dependencies intentionally on the original development machine or remove `--locked` only when you mean to refresh the lockfile.

## Updating packages intentionally

To update the lockfile after changing dependency constraints:

```bash
uv lock --upgrade
uv sync
```

Review the lockfile diff before committing. Do not update dependencies as part of a benchmark run unless the version change is the experiment being tested.

## Local venv vs Modal image

The local `.venv` and the remote Modal container are different environments.

The local `.venv` should contain packages needed to run local scripts and talk to Modal, such as `modal`, validation code, tests, and small CPU checks.

The Modal image should pin packages needed inside the remote worker, especially GPU/runtime-sensitive packages such as `sglang`, `torch`, CUDA-related wheels, tokenizer/model dependencies, and serving-specific tools. Those pins belong in the Modal app/image code, not only in the local venv.

For benchmark evidence, record both:

- local project revision and `uv.lock` state for the client/analysis environment
- Modal image package pins, model revision, GPU type, and runtime details for the serving environment

## Secrets

Do not store Modal tokens, Hugging Face tokens, API keys, `.env` files, or `.modal.toml` in Git. Keep credentials local or in Modal Secrets.
