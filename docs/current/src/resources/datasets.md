# Dataset Specification

A Dataset resource is a metadata reference to a data source. The actual data lives on HuggingFace Hub or the local filesystem. On import, Studio extracts feature column names and stores them as a JSON string, enabling downstream prompt template construction without reloading the data.

**Source**: `ft/datasets.py`, `ft/db/model.py`

## Supported Types

| Type | Source | Identifier Field | Feature Extraction Method |
|---|---|---|---|
| `huggingface` | HuggingFace Hub | `huggingface_name` | `load_dataset_builder()` -> `info.features.keys()` |
| `project` | Local HF-compatible directory | `location` | Not extracted |
| `project_csv` | Local CSV file | `location` | Read header row via `csv.reader` |
| `project_json` | Local JSON file | `location` | Read first object keys via `json.load` |
| `project_jsonl` | Local JSONL file | `location` | Read first line keys via `json.loads` |

## ORM Schema

```python
class Dataset(Base, MappedProtobuf, MappedDict):
    __tablename__ = "datasets"
    id = Column(String, primary_key=True)    # UUID
    type = Column(String)                     # DatasetType enum value
    name = Column(String)                     # Display name
    description = Column(Text)                # Auto-populated for HF datasets
    huggingface_name = Column(String)         # HF Hub identifier (HF type only)
    location = Column(Text)                   # Filesystem path (project types only)
    features = Column(Text)                   # JSON-serialized list of column names
```

## Import Validation

`add_dataset()` dispatches to type-specific validators before creating a record:

**All types**:
- `type` field is required.
- Duplicate detection by name (local types) or `huggingface_name` (HF type).

**HuggingFace** (`_validate_huggingface_dataset_request`):
- `huggingface_name` field required and non-blank.
- Validates dataset exists on Hub via `load_dataset_builder()`.
- Extracts `dataset_info.features.keys()` for feature list.
- Stores `dataset_info.description` as the description.

**CSV** (`_validate_local_csv_dataset_request`):
- `location` field required, must end with `.csv`.
- `name` field required and non-blank.
- Reads header row with `csv.reader(file)` / `next(reader)` for features.

**JSON** (`_validate_local_json_dataset_request`):
- `location` field required, must end with `.json`.
- Reads first object in the JSON array for feature keys.

**JSONL** (`_validate_local_jsonl_dataset_request`):
- `location` field required, must end with `.jsonl`.
- Reads first line, parses as JSON, extracts keys for features.

## Feature Extraction Functions

```python
extract_features_from_csv(location)   # csv.reader -> next(reader)
extract_features_from_json(location)  # json.load -> next(iter(data)).keys()
extract_features_from_jsonl(location) # json.loads(first_line).keys()
```

Features are stored as `json.dumps(features)` in the `features` column. Downstream consumers (prompt templates, training scripts) parse this back with `json.loads()`.

## Loading into Memory

`load_dataset_into_memory(dataset: DatasetMetadata)` normalizes all dataset types into a HuggingFace `DatasetDict` with at minimum a `train` key:

| Type | Load Method | Wrapping |
|---|---|---|
| `huggingface` | `datasets.load_dataset(huggingface_name)` | Already a `DatasetDict` |
| `project_csv` | `datasets.load_dataset('csv', data_files=location)` | Already a `DatasetDict` |
| `project_json` | `datasets.Dataset.from_json(location)` | Wrapped in `DatasetDict({'train': ds})` |
| `project_jsonl` | `datasets.Dataset.from_json(location)` | Wrapped in `DatasetDict({'train': ds})` |

If the loaded object is a `Dataset` (not `DatasetDict`), it is wrapped: `DatasetDict({'train': ds})`.

## Removal

`remove_dataset()` deletes the Dataset record. If `request.remove_prompts` is set, also deletes all Prompt records with matching `dataset_id` via cascading delete.

## Protobuf Message

`DatasetMetadata` fields: `id`, `type`, `name`, `description`, `huggingface_name`, `location`, `features`.
