# Resource Concepts

Fine Tuning Studio manages seven resource types. All use UUID string primary keys generated via `uuid4()`. Resources are metadata entries stored in SQLite -- the actual artifacts (model weights, dataset files, adapter checkpoints) live on the filesystem, HuggingFace Hub, or the CML Model Registry.

## Resource Types

| Resource | Table | Purpose |
|---|---|---|
| **Dataset** | `datasets` | Reference to a HuggingFace Hub dataset or local file (CSV, JSON, JSONL) |
| **Model** | `models` | Base foundation model from HuggingFace Hub or CML Model Registry |
| **Adapter** | `adapters` | PEFT LoRA adapter -- produced by training, imported from disk, or fetched from Hub |
| **Prompt** | `prompts` | Format-string template mapping dataset features into training input |
| **Config** | `configs` | Named configuration blob (training args, BnB, LoRA, generation, Axolotl YAML) |
| **FineTuningJob** | `fine_tuning_jobs` | CML Job that trains a PEFT adapter |
| **EvaluationJob** | `evaluation_jobs` | CML Job that runs MLflow evaluation against model+adapter combinations |

## Entity Relationships

```d2
direction: right

Model: Model {
  shape: class
  id: String {constraint: PK}
  type: String
  name: String
  huggingface_model_name: String
}

Dataset: Dataset {
  shape: class
  id: String {constraint: PK}
  type: String
  name: String
  features: Text (JSON)
}

Adapter: Adapter {
  shape: class
  id: String {constraint: PK}
  type: String
  name: String
  model_id: String {constraint: FK}
}

Prompt: Prompt {
  shape: class
  id: String {constraint: PK}
  name: String
  dataset_id: String {constraint: FK}
  prompt_template: String
}

Config: Config {
  shape: class
  id: String {constraint: PK}
  type: String
  config: Text (JSON/YAML)
  is_default: Integer
}

FineTuningJob: FineTuningJob {
  shape: class
  id: String {constraint: PK}
  base_model_id: String {constraint: FK}
  dataset_id: String {constraint: FK}
  prompt_id: String {constraint: FK}
  adapter_id: String {constraint: FK}
  training_arguments_config_id: String {constraint: FK}
  model_bnb_config_id: String {constraint: FK}
  adapter_bnb_config_id: String {constraint: FK}
  lora_config_id: String {constraint: FK}
  axolotl_config_id: String {constraint: FK}
  user_config_id: String {constraint: FK}
}

EvaluationJob: EvaluationJob {
  shape: class
  id: String {constraint: PK}
  base_model_id: String {constraint: FK}
  dataset_id: String {constraint: FK}
  adapter_id: String {constraint: FK}
  model_bnb_config_id: String {constraint: FK}
  generation_config_id: String {constraint: FK}
}

Adapter -> Model: model_id
Prompt -> Dataset: dataset_id
FineTuningJob -> Model: base_model_id
FineTuningJob -> Dataset: dataset_id
FineTuningJob -> Prompt: prompt_id
FineTuningJob -> Adapter: adapter_id
FineTuningJob -> Config: config FKs (6)
EvaluationJob -> Model: base_model_id
EvaluationJob -> Dataset: dataset_id
EvaluationJob -> Adapter: adapter_id
EvaluationJob -> Config: config FKs (3)
```

## Type Enums

All type enums are defined in `ft/api/types.py` as `str, Enum` subclasses.

| Enum | Values |
|---|---|
| `DatasetType` | `huggingface`, `project`, `project_csv`, `project_json`, `project_jsonl` |
| `ModelType` | `huggingface`, `project`, `model_registry` |
| `AdapterType` | `project`, `huggingface`, `model_registry` |
| `PromptType` | `in_place` |
| `ConfigType` | `training_arguments`, `bitsandbytes_config`, `generation_config`, `lora_config`, `custom`, `axolotl`, `axolotl_dataset_formats` |
| `FineTuningFrameworkType` | `legacy`, `axolotl` |
| `ModelExportType` | `model_registry`, `cml_model` |
| `EvaluationJobType` | `mlflow` |
| `ModelFrameworkType` | `pytorch`, `tensorflow`, `onnx` |

## ORM Layer

All ORM models inherit from `sqlalchemy.orm.declarative_base()` plus two mixins defined in `ft/db/model.py`:

**`MappedProtobuf`** -- bidirectional protobuf conversion:
- `from_message(message)` -- class method. Extracts set fields from a protobuf message via `ListFields()` and passes them as kwargs to the ORM constructor.
- `to_protobuf(protobuf_cls)` -- instance method. Converts non-null ORM columns into a protobuf message by matching field names.

**`MappedDict`** -- bidirectional dict conversion:
- `from_dict(d)` -- class method. Constructs an ORM instance from a plain dictionary.
- `to_dict()` -- instance method. Returns a dictionary of all non-null column values via SQLAlchemy `inspect()`.

The serialization chain for any resource:

```
Protobuf message  <-->  ORM model  <-->  Python dict
     from_message() / to_protobuf()   from_dict() / to_dict()
```

## Table Registry

`ft/db/model.py` maintains two registries used by the database import/export subsystem:

```python
TABLE_TO_MODEL_REGISTRY = {
    'datasets': Dataset,
    'models': Model,
    'prompts': Prompt,
    'adapters': Adapter,
    'fine_tuning_jobs': FineTuningJob,
    'evaluation_jobs': EvaluationJob,
    'configs': Config,
}

MODEL_TO_TABLE_REGISTRY = {v: k for k, v in TABLE_TO_MODEL_REGISTRY.items()}
```

Any new resource type must be added to `TABLE_TO_MODEL_REGISTRY` for database import/export to function correctly.
