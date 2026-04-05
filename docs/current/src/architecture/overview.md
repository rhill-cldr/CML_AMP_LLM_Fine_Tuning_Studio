# System Overview

Fine Tuning Studio is a three-layer application running inside a single CML Application pod. A Streamlit frontend communicates with a gRPC backend over localhost; the backend persists metadata to SQLite and dispatches CML Jobs for training and evaluation workloads.

## Component Topology

```d2
direction: right

browser: Browser {
  shape: person
}

streamlit: Streamlit UI {
  label: "Streamlit UI\n(CDSW_APP_PORT)"
}

client: FineTuningStudioClient {
  label: "FineTuningStudioClient\n(gRPC stub wrapper)"
}

grpc: gRPC Server {
  label: "gRPC Server\n(port 50051)"
}

domain: Domain Modules {
  label: "Domain Modules\n(ft/datasets, ft/models, ft/adapters,\nft/prompts, ft/jobs, ft/evaluation,\nft/configs, ft/databse_ops)"
}

dao: FineTuningStudioDao {
  label: "DAO\n(SQLAlchemy)"
}

sqlite: SQLite {
  label: ".app/state.db"
  shape: cylinder
}

cmlapi: CML API {
  label: "cmlapi SDK"
}

cmljobs: CML Jobs {
  label: "CML Jobs\n(training / eval)"
  shape: hexagon
}

browser -> streamlit: HTTP
streamlit -> client
client -> grpc: gRPC (insecure channel)
grpc -> domain: method dispatch
domain -> dao: session queries
dao -> sqlite: SQL
domain -> cmlapi: job dispatch
cmlapi -> cmljobs: create/run
```

## Layer Summary

### Presentation Layer

Entry point: `main.py`. Page modules live in `pgs/`. Two navigation modes are controlled by the `IS_COMPOSABLE` environment variable:

- **Composable mode** (`IS_COMPOSABLE` set): Horizontal navbar with dropdown menus for Home, Database, Resources, Experiments, AI Workbench, Examples, and Feedback.
- **Standard mode** (default): Sidebar navigation with section headers and Material Design icons.

Pages obtain shared gRPC and CML client instances through `@st.cache_resource` decorators defined in `pgs/streamlit_utils.py`. See [Streamlit Presentation Layer](./streamlit-layer.md) for full details.

### Application Layer

A gRPC server runs on port 50051, started by `bin/start-grpc-server.py` as a background subprocess. The service class `FineTuningStudioApp` in `ft/service.py` implements `FineTuningStudioServicer` (generated from protobuf). It is a pure router -- each RPC method delegates to a domain function in the corresponding module:

| Module | Domain |
|---|---|
| `ft/datasets.py` | Dataset import, listing, removal |
| `ft/models.py` | Model registration, export |
| `ft/adapters.py` | Adapter management, dataset split lookup |
| `ft/prompts.py` | Prompt template CRUD |
| `ft/jobs.py` | Fine-tuning job dispatch and tracking |
| `ft/evaluation.py` | Evaluation job dispatch and tracking |
| `ft/configs.py` | Configuration blob management |
| `ft/databse_ops.py` | Database export/import operations |

The servicer holds a `cmlapi.default_client()` and a `FineTuningStudioDao` instance, passing both to every domain function call. See [gRPC Service Design](./grpc-service.md) for the full API surface.

### Data Layer

SQLite at `.app/state.db` via SQLAlchemy ORM. Seven tables: `models`, `datasets`, `adapters`, `prompts`, `fine_tuning_jobs`, `evaluation_jobs`, `configs`. The DAO manages sessions with connection pooling (`pool_size=5`, `max_overflow=10`, `pool_timeout=30`, `pool_recycle=1800`). See [Data Tier](./data-tier.md) for schemas and the DAO API.

## Initialization Sequence

The startup sequence is defined in `.project-metadata.yaml` and executed by `bin/start-app-script.sh`:

1. **Install dependencies** -- `bin/install-dependencies-uv.py` installs from `requirements.txt` and performs `pip install -e .` to install the `ft` package in dev mode.
2. **Create template CML Jobs** -- `Accel_Finetuning_Base_Job` and `Mlflow_Evaluation_Base_Job` are created as reusable job templates for fine-tuning and evaluation dispatch.
3. **Initialize project defaults** -- `bin/initialize-project-defaults-uv.py` populates default datasets, prompts, models, and adapters from `data/project_defaults.json`.
4. **Start gRPC server** -- `bin/start-grpc-server.py` launches as a background process (`&`), binds to port 50051 with a `ThreadPoolExecutor(max_workers=10)`, and sets `FINE_TUNING_SERVICE_IP` and `FINE_TUNING_SERVICE_PORT` as CML project environment variables via `cmlapi`.
5. **Start Streamlit** -- `uv run -m streamlit run main.py --server.port $CDSW_APP_PORT --server.address 127.0.0.1`.

Both processes (gRPC server and Streamlit) run in the same pod. The gRPC server is the subprocess; Streamlit is the foreground process that keeps the CML Application alive.

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `FINE_TUNING_SERVICE_IP` | gRPC server IP address | Set at startup from `CDSW_IP_ADDRESS` |
| `FINE_TUNING_SERVICE_PORT` | gRPC server port | `50051` |
| `FINE_TUNING_STUDIO_SQLITE_DB` | SQLite database file path | `.app/state.db` |
| `CDSW_PROJECT_ID` | CML project identifier | Set by CML runtime |
| `CDSW_APP_PORT` | Streamlit server port | Set by CML runtime |
| `HUGGINGFACE_ACCESS_TOKEN` | HuggingFace Hub token for gated models | Optional (empty string) |
| `IS_COMPOSABLE` | Enable horizontal navbar mode | Optional (unset = sidebar) |
| `CUSTOM_LORA_ADAPTERS_DIR` | Directory for custom LoRA adapters | `data/adapters/` |
| `FINE_TUNING_STUDIO_PROJECT_DEFAULTS` | Path to project defaults JSON | `data/project_defaults.json` |

## Key Takeaway for Harness Builders

The gRPC API is the sole interface to application logic. The Streamlit UI is one client of this API, not the source of truth. Any external harness, CLI tool, or automation script should instantiate a `FineTuningStudioClient` (or use the generated gRPC stub directly) and interact through the protobuf contract. The database is an implementation detail behind the DAO -- never access `.app/state.db` directly from external code.

To build a custom training harness:

1. Import `FineTuningStudioClient` from `ft.client`.
2. Register resources (datasets, models, prompts) via `Add*` RPCs.
3. Dispatch training via `StartFineTuningJob` with the desired resource IDs and compute configuration.
4. Poll job status via `GetFineTuningJob` or `ListFineTuningJobs`.
5. Evaluate results via `StartEvaluationJob`.

All resource IDs are UUIDs assigned by the service. Pass them by value between RPCs.
