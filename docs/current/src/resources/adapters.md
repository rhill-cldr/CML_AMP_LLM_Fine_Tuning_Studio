# Adapter Specification

An Adapter resource represents a PEFT LoRA adapter. Adapters are produced by fine-tuning jobs, imported from a local directory, or fetched from HuggingFace Hub. Each adapter is linked to a base model and optionally to the fine-tuning job and prompt template that produced it.

**Source**: `ft/adapters.py`, `ft/db/model.py`

## Supported Types

| Type | Source | Required Fields |
|---|---|---|
| `project` | Local directory | `location` (must exist as a directory) |
| `huggingface` | HuggingFace Hub | `huggingface_name` |
| `model_registry` | CML Model Registry | `cml_registered_model_id` |

## ORM Schema

```python
class Adapter(Base, MappedProtobuf, MappedDict):
    __tablename__ = "adapters"
    id = Column(String, primary_key=True)                                 # UUID
    type = Column(String)                                                  # AdapterType enum value
    name = Column(String)                                                  # Display name (unique)
    description = Column(String)
    huggingface_name = Column(String)                                      # HF Hub adapter identifier
    model_id = Column(String, ForeignKey('models.id'))                     # Base model FK
    location = Column(Text)                                                # Local path to adapter dir
    fine_tuning_job_id = Column(String, ForeignKey('fine_tuning_jobs.id')) # Producing job FK
    prompt_id = Column(String, ForeignKey('prompts.id'))                   # Training prompt FK
    cml_registered_model_id = Column(String)                               # CML Registry model ID
    mlflow_experiment_id = Column(String)                                  # MLflow experiment
    mlflow_run_id = Column(String)                                         # MLflow run
```

## Key Relationships

| FK Column | Target | Required |
|---|---|---|
| `model_id` | `models.id` | Yes -- the base model this adapter applies to |
| `fine_tuning_job_id` | `fine_tuning_jobs.id` | No -- only set for Studio-trained adapters |
| `prompt_id` | `prompts.id` | No -- only set for Studio-trained adapters |

## Import Validation

`_validate_add_adapter_request()` enforces:

1. **Required fields**: `name`, `model_id`, and `location` must all be present and non-blank.
2. **Directory existence**: `os.path.isdir(request.location)` must return `True`.
3. **Model FK**: `model_id` must reference an existing Model record.
4. **Unique name**: No existing adapter may share the same `name`.
5. **Optional FK checks**: If `fine_tuning_job_id` is provided, it must exist in `fine_tuning_jobs`. If `prompt_id` is provided, it must exist in `prompts`.

## Adapter Creation

`add_adapter()` validates the request, then creates an Adapter record with all provided fields mapped directly from the request.

## Dataset Split Tracking

`get_dataset_split_by_adapter()` retrieves the dataset fraction and train/test split used during training for a given adapter:

1. Joins `FineTuningJob` to `Adapter` on `adapter_name`.
2. If a matching job is found, returns its `dataset_fraction` and `train_test_split`.
3. If no matching job exists (imported adapter), returns defaults:

| Parameter | Default | Source |
|---|---|---|
| `dataset_fraction` | `1.0` | `TRAINING_DEFAULT_DATASET_FRACTION` |
| `train_test_split` | `0.9` | `TRAINING_DEFAULT_TRAIN_TEST_SPLIT` |

These defaults are defined in `ft/consts.py`.

## Protobuf Message

`AdapterMetadata` fields: `id`, `type`, `name`, `description`, `huggingface_name`, `model_id`, `location`, `fine_tuning_job_id`, `prompt_id`, `cml_registered_model_id`, `mlflow_experiment_id`, `mlflow_run_id`.
