# Data Tier

All Fine Tuning Studio metadata is persisted in a SQLite database at `.app/state.db` (configurable via `FINE_TUNING_STUDIO_SQLITE_DB`). The ORM layer uses SQLAlchemy declarative models defined in `ft/db/model.py`. Access is managed through `FineTuningStudioDao` in `ft/db/dao.py`.

## Schema Topology

```d2
direction: down

models: models {
  shape: sql_table
  id: String {constraint: PK}
  type: String
  framework: String
  name: String
  description: String
  huggingface_model_name: String
  location: String
  cml_registered_model_id: String
  mlflow_experiment_id: String
  mlflow_run_id: String
}

datasets: datasets {
  shape: sql_table
  id: String {constraint: PK}
  type: String
  name: String
  description: Text
  huggingface_name: String
  location: Text
  features: Text
}

prompts: prompts {
  shape: sql_table
  id: String {constraint: PK}
  type: String
  name: String
  description: String
  dataset_id: String {constraint: FK}
  prompt_template: String
  input_template: String
  completion_template: String
}

configs: configs {
  shape: sql_table
  id: String {constraint: PK}
  type: String
  description: String
  config: Text
  model_family: String
  is_default: Integer
}

adapters: adapters {
  shape: sql_table
  id: String {constraint: PK}
  type: String
  name: String
  description: String
  huggingface_name: String
  model_id: String {constraint: FK}
  location: Text
  fine_tuning_job_id: String {constraint: FK}
  prompt_id: String {constraint: FK}
  cml_registered_model_id: String
  mlflow_experiment_id: String
  mlflow_run_id: String
}

fine_tuning_jobs: fine_tuning_jobs {
  shape: sql_table
  id: String {constraint: PK}
  base_model_id: String {constraint: FK}
  dataset_id: String {constraint: FK}
  prompt_id: String {constraint: FK}
  adapter_id: String {constraint: FK}
  framework_type: String
  "...": "(20 more columns)"
}

evaluation_jobs: evaluation_jobs {
  shape: sql_table
  id: String {constraint: PK}
  base_model_id: String {constraint: FK}
  dataset_id: String {constraint: FK}
  prompt_id: String {constraint: FK}
  adapter_id: String {constraint: FK}
  "...": "(12 more columns)"
}

adapters -> models: model_id
adapters -> fine_tuning_jobs: fine_tuning_job_id
adapters -> prompts: prompt_id
prompts -> datasets: dataset_id
fine_tuning_jobs -> models: base_model_id
fine_tuning_jobs -> datasets: dataset_id
fine_tuning_jobs -> prompts: prompt_id
fine_tuning_jobs -> adapters: adapter_id
fine_tuning_jobs -> configs: "*_config_id (6 FKs)"
evaluation_jobs -> models: base_model_id
evaluation_jobs -> datasets: dataset_id
evaluation_jobs -> prompts: prompt_id
evaluation_jobs -> adapters: adapter_id
evaluation_jobs -> configs: "*_config_id (3 FKs)"
```

## Table Schemas

All primary keys are `String` type (UUIDs assigned by domain logic). All columns are nullable except `id`. ORM classes are defined in `ft/db/model.py`.

### models

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | PK, NOT NULL | UUID |
| `type` | String | | Source type (e.g., `huggingface`, `cml`) |
| `framework` | String | | Model framework identifier |
| `name` | String | | Display name |
| `description` | String | | Human-readable description |
| `huggingface_model_name` | String | | HuggingFace Hub model ID |
| `location` | String | | Local filesystem path |
| `cml_registered_model_id` | String | | CML Model Registry ID |
| `mlflow_experiment_id` | String | | Associated MLflow experiment |
| `mlflow_run_id` | String | | Associated MLflow run |

### datasets

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | PK, NOT NULL | UUID |
| `type` | String | | Source type (e.g., `huggingface`, `local`) |
| `name` | String | | Display name |
| `description` | Text | | Long-form description |
| `huggingface_name` | String | | HuggingFace Hub dataset ID |
| `location` | Text | | Local filesystem path |
| `features` | Text | | JSON string of dataset feature names |

### adapters

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | PK, NOT NULL | UUID |
| `type` | String | | Source type |
| `name` | String | | Display name |
| `description` | String | | Human-readable description |
| `huggingface_name` | String | | HuggingFace Hub adapter ID |
| `model_id` | String | FK -> `models.id` | Base model this adapter targets |
| `location` | Text | | Local filesystem path to adapter weights |
| `fine_tuning_job_id` | String | FK -> `fine_tuning_jobs.id` | Job that produced this adapter |
| `prompt_id` | String | FK -> `prompts.id` | Prompt template used during training |
| `cml_registered_model_id` | String | | CML Model Registry ID |
| `mlflow_experiment_id` | String | | Associated MLflow experiment |
| `mlflow_run_id` | String | | Associated MLflow run |

### prompts

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | PK, NOT NULL | UUID |
| `type` | String | | Prompt type |
| `name` | String | | Display name |
| `description` | String | | Human-readable description |
| `dataset_id` | String | FK -> `datasets.id` | Dataset this prompt is designed for |
| `prompt_template` | String | | Full prompt format string |
| `input_template` | String | | Input portion template |
| `completion_template` | String | | Completion portion template |

### fine_tuning_jobs

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | PK, NOT NULL | UUID |
| `base_model_id` | String | FK -> `models.id` | Base model to fine-tune |
| `dataset_id` | String | FK -> `datasets.id` | Training dataset |
| `prompt_id` | String | FK -> `prompts.id` | Prompt template |
| `num_workers` | Integer | | Number of worker processes |
| `cml_job_id` | String | | CML Job ID for tracking |
| `adapter_id` | String | FK -> `adapters.id` | Resulting adapter |
| `num_cpu` | Integer | | CPU allocation |
| `num_gpu` | Integer | | GPU allocation |
| `num_memory` | Integer | | Memory allocation (GB) |
| `num_epochs` | Integer | | Training epochs |
| `learning_rate` | Double | | Learning rate |
| `out_dir` | String | | Output directory for adapter weights |
| `training_arguments_config_id` | String | FK -> `configs.id` | Training arguments config |
| `model_bnb_config_id` | String | FK -> `configs.id` | Model BitsAndBytes quantization config |
| `adapter_bnb_config_id` | String | FK -> `configs.id` | Adapter BitsAndBytes quantization config |
| `lora_config_id` | String | FK -> `configs.id` | LoRA hyperparameters config |
| `training_arguments_config` | String | | Serialized training arguments (snapshot) |
| `model_bnb_config` | String | | Serialized model BnB config (snapshot) |
| `adapter_bnb_config` | String | | Serialized adapter BnB config (snapshot) |
| `lora_config` | String | | Serialized LoRA config (snapshot) |
| `dataset_fraction` | Double | | Fraction of dataset to use |
| `train_test_split` | Double | | Train/test split ratio |
| `user_script` | String | | Custom user training script path |
| `user_config_id` | String | FK -> `configs.id` | Custom user config |
| `framework_type` | String | | Training framework (`legacy`, `axolotl`, etc.) |
| `axolotl_config_id` | String | FK -> `configs.id` | Axolotl YAML config |
| `gpu_label_id` | Integer | | GPU label selector |
| `adapter_name` | String | | Name assigned to the output adapter |

The `fine_tuning_jobs` table stores both config ID references (foreign keys to `configs`) and serialized config snapshots (plain string columns). This allows job records to remain self-describing even if the referenced config is later deleted.

### evaluation_jobs

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | PK, NOT NULL | UUID |
| `type` | String | | Evaluation type |
| `cml_job_id` | String | | CML Job ID for tracking |
| `parent_job_id` | String | | Parent fine-tuning job (if derived) |
| `base_model_id` | String | FK -> `models.id` | Model under evaluation |
| `dataset_id` | String | FK -> `datasets.id` | Evaluation dataset |
| `prompt_id` | String | FK -> `prompts.id` | Prompt template |
| `num_workers` | Integer | | Number of worker processes |
| `adapter_id` | String | FK -> `adapters.id` | Adapter under evaluation |
| `num_cpu` | Integer | | CPU allocation |
| `num_gpu` | Integer | | GPU allocation |
| `num_memory` | Integer | | Memory allocation (GB) |
| `evaluation_dir` | String | | Output directory for evaluation artifacts |
| `model_bnb_config_id` | String | FK -> `configs.id` | Model BnB quantization config |
| `adapter_bnb_config_id` | String | FK -> `configs.id` | Adapter BnB quantization config |
| `generation_config_id` | String | FK -> `configs.id` | Generation config for inference |
| `model_bnb_config` | String | | Serialized model BnB config (snapshot) |
| `adapter_bnb_config` | String | | Serialized adapter BnB config (snapshot) |
| `generation_config` | String | | Serialized generation config (snapshot) |

### configs

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | PK, NOT NULL | UUID |
| `type` | String | | Config type (`training_arguments`, `bnb`, `lora`, `generation`, `axolotl`) |
| `description` | String | | Human-readable description |
| `config` | Text | | JSON or YAML content stored as string |
| `model_family` | String | | Model family this config targets |
| `is_default` | Integer | | `1` = shipped default, `0` = user-created |

## ORM Mix-ins

All ORM model classes inherit from three bases: `Base` (SQLAlchemy declarative base), `MappedProtobuf`, and `MappedDict`. These mix-ins provide bidirectional serialization.

### MappedProtobuf

Converts between protobuf messages and ORM instances.

```python
# Protobuf message -> ORM instance
adapter_orm = Adapter.from_message(adapter_proto_msg)

# ORM instance -> Protobuf message
adapter_proto = adapter_orm.to_protobuf(AdapterMetadata)
```

`from_message()` uses `ListFields()` (protobuf >= 3.15) to extract only fields that were explicitly set in the message, avoiding default-value contamination. `to_protobuf()` iterates the ORM instance's non-null columns and sets matching fields on a new protobuf message.

### MappedDict

Converts between Python dictionaries and ORM instances.

```python
# Dict -> ORM instance
model_orm = Model.from_dict({"id": "abc", "name": "llama-2"})

# ORM instance -> Dict (non-null fields only)
model_dict = model_orm.to_dict()
```

### Table-Model Registry

`ft/db/model.py` exports two lookup dictionaries for programmatic table access:

```python
TABLE_TO_MODEL_REGISTRY = {
    'datasets': Dataset,
    'models': Model,
    'prompts': Prompt,
    'adapters': Adapter,
    'fine_tuning_jobs': FineTuningJob,
    'evaluation_jobs': EvaluationJob,
    'configs': Config
}

MODEL_TO_TABLE_REGISTRY = {v: k for k, v in TABLE_TO_MODEL_REGISTRY.items()}
```

These are used by the database import/export logic to iterate all application tables.

## DAO

`FineTuningStudioDao` in `ft/db/dao.py` manages SQLAlchemy engine and session lifecycle.

### Constructor

```python
class FineTuningStudioDao:
    def __init__(self, engine_url=None, echo=False, engine_args={}):
        if engine_url is None:
            engine_url = f"sqlite+pysqlite:///{get_sqlite_db_location()}"
        self.engine = create_engine(engine_url, echo=echo, **engine_args)
        self.Session = sessionmaker(bind=self.engine, autoflush=True, autocommit=False)
        Base.metadata.create_all(self.engine)
```

The servicer instantiates the DAO with connection pool parameters:

| Parameter | Value | Description |
|---|---|---|
| `pool_size` | 5 | Persistent connections in the pool |
| `max_overflow` | 10 | Additional connections beyond pool_size |
| `pool_timeout` | 30 | Seconds to wait for a connection |
| `pool_recycle` | 1800 | Seconds before a connection is recycled |

Tables are auto-created on first initialization via `Base.metadata.create_all(engine)`.

### Session Context Manager

All domain functions access the database through `dao.get_session()`:

```python
@contextmanager
def get_session(self):
    session = self.Session()
    try:
        yield session
        session.commit()
    except Exception as e:
        session.rollback()
        raise e
    finally:
        session.close()
```

Usage in domain code:

```python
def list_datasets(request, cml, dao):
    with dao.get_session() as session:
        datasets = session.query(Dataset).all()
        # ... convert and return
```

The context manager guarantees: commit on success, rollback on exception, close in all cases.

## Database Export and Import

`ft/db/db_import_export.py` provides `DatabaseJsonConverter` for full database serialization.

### Export

`export_to_json(output_path=None)` iterates all non-system tables (excluding `sqlite_*` internal tables), captures the `CREATE TABLE` schema and all row data, and returns a JSON string:

```json
{
  "models": {
    "schema": "CREATE TABLE IF NOT EXISTS models (...)",
    "data": [
      {"id": "abc-123", "name": "llama-2", "type": "huggingface", ...}
    ]
  },
  "datasets": { ... },
  ...
}
```

If `output_path` is provided, the JSON is also written to that file.

### Import

`import_from_json(json_path)` reads a JSON file in the export format, executes each table's `CREATE TABLE IF NOT EXISTS` statement, and inserts all rows. Rows that fail to insert (e.g., due to duplicate primary keys) are logged but do not abort the import.

## Alembic Migrations

Schema migrations are managed by Alembic. Configuration is at `alembic.ini` with migration scripts in `db_migrations/`. When adding or modifying columns, generate a new migration with:

```bash
alembic revision --autogenerate -m "description of change"
alembic upgrade head
```

The DAO's `create_all()` call handles initial table creation, but column additions and type changes on existing databases require Alembic migrations.

## Cross-References

- [System Overview](./overview.md) -- initialization sequence and environment variables
- [gRPC Service Design](./grpc-service.md) -- how domain functions receive the DAO
- [Configuration Specification](../resources/configs.md) -- config type taxonomy and validation
