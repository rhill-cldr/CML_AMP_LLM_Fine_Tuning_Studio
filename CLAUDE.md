# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

CML Fine Tuning Studio is a Cloudera AMP (Applied ML Prototype) for managing, fine-tuning, and evaluating LLMs within Cloudera Machine Learning. It orchestrates CML Jobs for training and evaluation via a three-layer architecture.

## Architecture

**Three-layer design:**

1. **Presentation Layer** — Streamlit UI (`main.py` entry point, pages in `pgs/`). Pages call the gRPC service via `FineTuningStudioClient` (`ft/client.py`).
2. **Application Layer** — gRPC server (`bin/start-grpc-server.py`, port 50051). Service implementation in `ft/service.py` (implements `FineTuningStudioServicer`). Delegates to domain modules: `ft/datasets.py`, `ft/models.py`, `ft/adapters.py`, `ft/prompts.py`, `ft/jobs.py`, `ft/evaluation.py`, `ft/configs.py`, `ft/export.py`.
3. **Data Layer** — SQLite (`.app/state.db`) via SQLAlchemy ORM. Models in `ft/db/model.py`, DAO in `ft/db/dao.py`. Alembic migrations in `db_migrations/`.

**Key data flow:** Streamlit page → `FineTuningStudioClient` (gRPC) → `FineTuningStudioApp` service → domain module → `FineTuningStudioDao` → SQLite

**API surface** is defined in `ft/proto/fine_tuning_studio.proto`. Protobuf ↔ ORM conversion uses `MappedProtobuf`/`MappedDict` base classes in `ft/db/model.py`.

## Common Commands

```bash
# Run tests with coverage
./bin/run-tests.sh
# Equivalent to: pytest -v --cov=ft --cov-report=html --cov-report=xml -s tests/

# Run a single test file
pytest -v -s tests/test_datasets.py

# Run a single test
pytest -v -s tests/test_datasets.py::TestDatasets::test_add_dataset

# Format code (autoflake + autopep8)
./bin/format.sh

# Regenerate protobuf Python files after changing .proto
./bin/generate-proto-python.sh
```

## Code Style

- Max line length: 120 (autopep8 aggressive=3)
- Formatting: `autoflake` removes unused imports, `autopep8` enforces PEP 8
- Always run `./bin/format.sh` before committing
- Tests use `pytest` + `unittest` (unittest.mock.patch for mocking)

## Key Environment Variables

| Variable | Purpose |
|---|---|
| `FINE_TUNING_STUDIO_SQLITE_DB` | Override SQLite DB path (default: `.app/state.db`) |
| `FINE_TUNING_STUDIO_PROJECT_DEFAULTS` | Path to project defaults JSON |
| `HUGGINGFACE_ACCESS_TOKEN` | HF Hub access for gated models |
| `FINE_TUNING_SERVICE_IP` / `FINE_TUNING_SERVICE_PORT` | gRPC server location (set at runtime) |
| `CUSTOM_LORA_ADAPTERS_DIR` | Custom adapters directory |
| `IS_COMPOSABLE` | Composable layout flag for Streamlit navbar |

## Key Directories

- `ft/` — Core application package (service, domain logic, DB, proto, training, eval)
- `ft/scripts/` — Base script templates for fine-tuning and evaluation CML Jobs
- `ft/config/axolotl/` — Axolotl training config templates and dataset format configs
- `pgs/` — Streamlit UI pages (19 pages covering datasets, models, training, eval, export)
- `bin/` — Build scripts, server startup, formatting, test runner
- `tests/` — Test suite (13 test modules mirroring `ft/` structure)
- `data/` — Default datasets, adapters, models, project defaults JSON
- `db_migrations/` — Alembic migration scripts

## Testing Notes

- CI runs on Python 3.11 against `main` and `dev` branches (`.github/workflows/run-tests.yaml`)
- Coverage threshold: >10%
- Test modules mirror source: `test_datasets.py`, `test_models.py`, `test_jobs.py`, etc.
- Heavy use of `unittest.mock.patch` to mock CML API, gRPC, and DB interactions

## Developer's Guide

The mdbook-based Developer's Guide lives in `docs/current/` and is deployed to GitHub Pages via `.github/workflows/docs.yml`. It covers architecture reference, resource specifications (datasets, models, adapters, prompts, configs), training/evaluation job lifecycles, deployment artifacts, and validation rules. Build locally with `mdbook build` from `docs/current/`.

Published at: https://rhill-cldr.github.io/CML_AMP_LLM_Fine_Tuning_Studio/

## PR Workflow

- PRs target the `dev` branch (active development branch), not `main`
- Run `./bin/format.sh` and `./bin/run-tests.sh` before opening a PR
