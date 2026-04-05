# gRPC Service Design

The Fine Tuning Studio API is defined as a single gRPC service in `ft/proto/fine_tuning_studio.proto`. The service exposes 29 RPCs organized by resource domain. A generated Python stub provides the transport layer; `FineTuningStudioClient` wraps it with error handling and convenience methods.

## Service Architecture

```d2
direction: down

proto: fine_tuning_studio.proto {
  label: "Protobuf Definition\n(ft/proto/fine_tuning_studio.proto)"
  shape: document
}

codegen: Code Generation {
  label: "bin/generate-proto-python.sh\n→ _pb2.py, _pb2_grpc.py"
}

stub: FineTuningStudioStub {
  label: "Generated Stub\n(ft/proto/fine_tuning_studio_pb2_grpc)"
}

client: FineTuningStudioClient {
  label: "Client Wrapper\n(ft/client.py)"
}

servicer: FineTuningStudioApp {
  label: "Servicer Implementation\n(ft/service.py)"
}

domain: Domain Functions {
  datasets: ft/datasets.py
  models: ft/models.py
  adapters: ft/adapters.py
  prompts: ft/prompts.py
  jobs: ft/jobs.py
  evaluation: ft/evaluation.py
  configs: ft/configs.py
  databse_ops: ft/databse_ops.py
}

proto -> codegen: grpc_tools.protoc
codegen -> stub
client -> stub: wraps
servicer -> domain: delegates each RPC
```

## RPC Catalog

Every domain follows the same pattern: `List`, `Get`, `Add` (or `Start` for jobs), and `Remove`. Request and response types use the naming convention `{Action}{Domain}Request` / `{Action}{Domain}Response`.

### Dataset RPCs

| RPC | Request Type | Response Type | Description |
|---|---|---|---|
| `ListDatasets` | `ListDatasetsRequest` | `ListDatasetsResponse` | Return all registered datasets |
| `GetDataset` | `GetDatasetRequest` | `GetDatasetResponse` | Return a single dataset by ID |
| `AddDataset` | `AddDatasetRequest` | `AddDatasetResponse` | Register a HuggingFace or local dataset |
| `RemoveDataset` | `RemoveDatasetRequest` | `RemoveDatasetResponse` | Delete a dataset registration |
| `GetDatasetSplitByAdapter` | `GetDatasetSplitByAdapterRequest` | `GetDatasetSplitByAdapterResponse` | Get dataset split info for a specific adapter |

### Model RPCs

| RPC | Request Type | Response Type | Description |
|---|---|---|---|
| `ListModels` | `ListModelsRequest` | `ListModelsResponse` | Return all registered models |
| `GetModel` | `GetModelRequest` | `GetModelResponse` | Return a single model by ID |
| `AddModel` | `AddModelRequest` | `AddModelResponse` | Register a HuggingFace or CML model |
| `ExportModel` | `ExportModelRequest` | `ExportModelResponse` | Export a model to CML Model Registry |
| `RemoveModel` | `RemoveModelRequest` | `RemoveModelResponse` | Delete a model registration |

### Adapter RPCs

| RPC | Request Type | Response Type | Description |
|---|---|---|---|
| `ListAdapters` | `ListAdaptersRequest` | `ListAdaptersResponse` | Return all registered adapters |
| `GetAdapter` | `GetAdapterRequest` | `GetAdapterResponse` | Return a single adapter by ID |
| `AddAdapter` | `AddAdapterRequest` | `AddAdapterResponse` | Register a local or HuggingFace adapter |
| `RemoveAdapter` | `RemoveAdapterRequest` | `RemoveAdapterResponse` | Delete an adapter registration |

### Prompt RPCs

| RPC | Request Type | Response Type | Description |
|---|---|---|---|
| `ListPrompts` | `ListPromptsRequest` | `ListPromptsResponse` | Return all prompt templates |
| `GetPrompt` | `GetPromptRequest` | `GetPromptResponse` | Return a single prompt by ID |
| `AddPrompt` | `AddPromptRequest` | `AddPromptResponse` | Create a new prompt template |
| `RemovePrompt` | `RemovePromptRequest` | `RemovePromptResponse` | Delete a prompt template |

### Fine-Tuning RPCs

| RPC | Request Type | Response Type | Description |
|---|---|---|---|
| `ListFineTuningJobs` | `ListFineTuningJobsRequest` | `ListFineTuningJobsResponse` | Return all fine-tuning jobs |
| `GetFineTuningJob` | `GetFineTuningJobRequest` | `GetFineTuningJobResponse` | Return a single job by ID |
| `StartFineTuningJob` | `StartFineTuningJobRequest` | `StartFineTuningJobResponse` | Dispatch a new fine-tuning CML Job |
| `RemoveFineTuningJob` | `RemoveFineTuningJobRequest` | `RemoveFineTuningJobResponse` | Delete a fine-tuning job record |

### Evaluation RPCs

| RPC | Request Type | Response Type | Description |
|---|---|---|---|
| `ListEvaluationJobs` | `ListEvaluationJobsRequest` | `ListEvaluationJobsResponse` | Return all evaluation jobs |
| `GetEvaluationJob` | `GetEvaluationJobRequest` | `GetEvaluationJobResponse` | Return a single evaluation job by ID |
| `StartEvaluationJob` | `StartEvaluationJobRequest` | `StartEvaluationJobResponse` | Dispatch a new evaluation CML Job |
| `RemoveEvaluationJob` | `RemoveEvaluationJobRequest` | `RemoveEvaluationJobResponse` | Delete an evaluation job record |

### Config RPCs

| RPC | Request Type | Response Type | Description |
|---|---|---|---|
| `ListConfigs` | `ListConfigsRequest` | `ListConfigsResponse` | Return all configuration blobs |
| `GetConfig` | `GetConfigRequest` | `GetConfigResponse` | Return a single config by ID |
| `AddConfig` | `AddConfigRequest` | `AddConfigResponse` | Create a new configuration |
| `RemoveConfig` | `RemoveConfigRequest` | `RemoveConfigResponse` | Delete a configuration |

### Database RPCs

| RPC | Request Type | Response Type | Description |
|---|---|---|---|
| `ExportDatabase` | `ExportDatabaseRequest` | `ExportDatabaseResponse` | Export entire database as JSON |
| `ImportDatabase` | `ImportDatabaseRequest` | `ImportDatabaseResponse` | Import database from JSON file |

## Servicer Implementation

`FineTuningStudioApp` in `ft/service.py` extends the generated `FineTuningStudioServicer`. It holds two shared resources initialized in `__init__`:

```python
class FineTuningStudioApp(FineTuningStudioServicer):
    def __init__(self):
        self.cml = cmlapi.default_client()
        self.dao = FineTuningStudioDao(engine_args={
            "pool_size": 5,
            "max_overflow": 10,
            "pool_timeout": 30,
            "pool_recycle": 1800,
        })
        self.project_id = os.getenv("CDSW_PROJECT_ID")
```

Every RPC method is a one-line delegation to the corresponding domain function, passing `(request, self.cml, self.dao)`:

```python
def ListDatasets(self, request, context):
    return list_datasets(request, self.cml, self.dao)

def StartFineTuningJob(self, request, context):
    return start_fine_tuning_job(request, self.cml, dao=self.dao)
```

Config and database RPCs omit the `cml` parameter since they operate on local data only.

## Client Wrapper

`FineTuningStudioClient` in `ft/client.py` wraps the generated stub with automatic error handling. On construction, it introspects all callable methods on the stub and wraps each one to convert `grpc.RpcError` into `ValueError` with cleaned messages.

```python
class FineTuningStudioClient:
    def __init__(self, server_ip=None, server_port=None):
        if not server_ip:
            server_ip = os.getenv("FINE_TUNING_SERVICE_IP")
        if not server_port:
            server_port = os.getenv("FINE_TUNING_SERVICE_PORT")
        self.channel = grpc.insecure_channel(f"{server_ip}:{server_port}")
        self.stub = FineTuningStudioStub(self.channel)

        # Auto-wrap all stub methods with error handling
        for attr in dir(self.stub):
            if not attr.startswith('_') and callable(getattr(self.stub, attr)):
                setattr(self, attr, self._grpc_error_handler(getattr(self.stub, attr)))
```

### Convenience Methods

The client provides shorthand accessors that construct the request internally:

| Method | Returns | Equivalent RPC |
|---|---|---|
| `get_datasets()` | `List[DatasetMetadata]` | `ListDatasets(ListDatasetsRequest()).datasets` |
| `get_models()` | `List[ModelMetadata]` | `ListModels(ListModelsRequest()).models` |
| `get_adapters()` | `List[AdapterMetadata]` | `ListAdapters(ListAdaptersRequest()).adapters` |
| `get_prompts()` | `List[PromptMetadata]` | `ListPrompts(ListPromptsRequest()).prompts` |
| `get_fine_tuning_jobs()` | `List[FineTuningJobMetadata]` | `ListFineTuningJobs(ListFineTuningJobsRequest()).fine_tuning_jobs` |
| `get_evaluation_jobs()` | `List[EvaluationJobMetadata]` | `ListEvaluationJobs(ListEvaluationJobsRequest()).evaluation_jobs` |

## Usage Example

```python
from ft.client import FineTuningStudioClient
from ft.api import *

client = FineTuningStudioClient()

# List all datasets
datasets = client.get_datasets()

# Add a HuggingFace dataset
client.AddDataset(AddDatasetRequest(
    type="huggingface",
    huggingface_name="tatsu-lab/alpaca",
    name="Alpaca"
))

# Start a fine-tuning job
client.StartFineTuningJob(StartFineTuningJobRequest(
    base_model_id="model-uuid",
    dataset_id="dataset-uuid",
    prompt_id="prompt-uuid",
    adapter_name="my-adapter",
    num_cpu=2,
    num_gpu=1,
    num_memory=16,
    framework_type="legacy"
))
```

All request and response types are importable from `ft.api`, which re-exports the generated protobuf classes.

## Protobuf Regeneration

After modifying `ft/proto/fine_tuning_studio.proto`, regenerate the Python bindings:

```bash
./bin/generate-proto-python.sh
```

This produces `ft/proto/fine_tuning_studio_pb2.py` (message classes) and `ft/proto/fine_tuning_studio_pb2_grpc.py` (stub and servicer base class). Both are checked into the repository. Do not edit them by hand.

## Server Startup

The gRPC server is started by `bin/start-grpc-server.py`:

1. Creates a `grpc.server` with `ThreadPoolExecutor(max_workers=10)`.
2. Registers `FineTuningStudioApp()` as the servicer.
3. Binds to `[::]:50051` (all interfaces).
4. Updates CML project environment variables (`FINE_TUNING_SERVICE_IP`, `FINE_TUNING_SERVICE_PORT`) via `cmlapi` so that any workload in the project can locate the server.
5. Blocks on `server.wait_for_termination()`.

The server process is launched as a background subprocess by `bin/start-app-script.sh` before Streamlit starts. See [System Overview](./overview.md) for the full initialization sequence.
