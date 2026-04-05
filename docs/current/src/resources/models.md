# Model Specification

A Model resource represents a base foundation model registered in the Studio's metadata store. Models serve as the starting point for fine-tuning and evaluation. The actual model weights are never stored by Studio -- they are downloaded at training time from HuggingFace Hub or resolved from the CML Model Registry.

**Source**: `ft/models.py`, `ft/db/model.py`, `ft/config/model_configs/config_loader.py`

## Supported Types

| Type | Source | Required Fields | Validation |
|---|---|---|---|
| `huggingface` | HuggingFace Hub | `huggingface_model_name` | `HfApi().model_info()` must succeed |
| `model_registry` | CML Model Registry | `model_registry_id` (request) | Fetches `RegisteredModel` via `cmlapi` |
| `project` | Local directory | `location` | Not yet fully implemented |

## ORM Schema

```python
class Model(Base, MappedProtobuf, MappedDict):
    __tablename__ = "models"
    id = Column(String, primary_key=True)            # UUID
    type = Column(String)                             # ModelType enum value
    framework = Column(String)                        # ModelFrameworkType (pytorch, tensorflow, onnx)
    name = Column(String)                             # Display name
    description = Column(String)
    huggingface_model_name = Column(String)           # HF Hub model identifier
    location = Column(String)                         # Local path (project type)
    cml_registered_model_id = Column(String)          # CML Registry model ID
    mlflow_experiment_id = Column(String)             # MLflow experiment (registry type)
    mlflow_run_id = Column(String)                    # MLflow run (registry type)
```

## Import Flow

`add_model()` validates and creates a Model record based on type:

**HuggingFace**:
1. Validate `huggingface_name` is non-empty and not already registered (duplicate check by `huggingface_model_name`).
2. Call `HfApi().model_info(name)` to confirm model exists on Hub.
3. Create Model with `type=HUGGINGFACE`, `name` and `huggingface_model_name` set to the stripped input.

**Model Registry**:
1. `model_registry_id` must be provided on the request.
2. Fetch `RegisteredModel` via `cml.get_registered_model(id)`.
3. Extract the first version's metadata: `registered_model.model_versions[0].model_version_metadata.mlflow_metadata`.
4. Create Model with `type=MODEL_REGISTRY`, `name` from `registered_model.name`, and populate `cml_registered_model_id`, `mlflow_experiment_id`, `mlflow_run_id`.

## Model Family Detection

`ft/config/model_configs/config_loader.py` provides `ModelMetadataFinder`:

```python
class ModelMetadataFinder:
    def __init__(self, model_name_or_path):
        self.model_name_or_path = model_name_or_path

    def fetch_model_family_from_config(self):
        config = AutoConfig.from_pretrained(self.model_name_or_path)
        return config.architectures[0]  # e.g., "LlamaForCausalLM"
```

This is used in two places:
- **Config filtering**: `list_configs()` filters default configs to those matching the model's architecture family.
- **Config creation**: `add_config()` with a `description` field uses `transform_name_to_family()` to resolve the model family for deduplication scoping.

Additional static methods:
- `fetch_bos_token_id_from_config(model_name_or_path)` -- returns `config.bos_token_id` (default: 1).
- `fetch_eos_token_id_from_config(model_name_or_path)` -- returns `config.eos_token_id` (default: 2).

## Export Routes

`export_model()` dispatches based on `ModelExportType`:

| Export Type | Handler | Target |
|---|---|---|
| `model_registry` | `export_model_registry_model()` | MLflow model registry |
| `cml_model` | `deploy_cml_model()` | CML Model endpoint |

Both handlers are defined in `ft/export.py`.

## Protobuf Message

`ModelMetadata` fields: `id`, `type`, `framework`, `name`, `huggingface_model_name`, `location`, `cml_registered_model_id`, `mlflow_experiment_id`, `mlflow_run_id`.
