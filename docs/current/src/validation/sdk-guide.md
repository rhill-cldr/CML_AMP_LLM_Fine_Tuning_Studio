# Building a Validation SDK

This chapter provides guidance for building a validation SDK that validates resources and job parameters before submitting them to the Fine Tuning Studio gRPC API. Pre-submission validation catches errors locally, avoiding round-trips to the server and failed CML Job launches.

**Source**: `ft/client.py`, `ft/proto/fine_tuning_studio_pb2.py`, `ft/api/types.py`

## Install the `ft` Package

The Fine Tuning Studio ships as a pip-installable package. Install it to get access to protobuf definitions, API types, and the gRPC client:

```bash
pip install -e /path/to/CML_AMP_LLM_Fine_Tuning_Studio
```

This provides the `ft` package in development mode. All protobuf-generated classes and enum types are available for import.

## Import Protobuf Types

```python
from ft.api import *

# Or import specific types:
from ft.proto.fine_tuning_studio_pb2 import (
    StartFineTuningJobRequest,
    AddDatasetRequest,
    AddModelRequest,
    AddConfigRequest,
)
from ft.api.types import (
    DatasetType,
    ConfigType,
    FineTuningFrameworkType,
)
```

These types define the exact field names, types, and enum values accepted by the gRPC API.

## Validation Architecture

Validation rules fall into two categories:

| Category | Requires | Examples |
|---|---|---|
| Local validation | No external dependencies | Regex checks, numeric bounds, enum membership, cross-field consistency |
| DB-dependent validation | `FineTuningStudioClient` connection | Foreign-key existence (model, dataset, prompt, adapter) |

A validation SDK should implement local validation in pure functions and DB-dependent validation through the client.

## Local Validation

Replicate the rules from the [Validation Rules Reference](./rules.md) that require no database access:

```python
import re

def validate_adapter_name(name: str) -> list[str]:
    """FT-002: adapter_name must be alphanumeric + hyphens."""
    errors = []
    if not re.match(r'^[a-zA-Z0-9-]+$', name):
        errors.append("FT-002: adapter_name must match ^[a-zA-Z0-9-]+$")
    return errors

def validate_resource_allocation(num_cpu: int, num_gpu: int, num_memory: int) -> list[str]:
    """FT-004, FT-005, FT-006: resource bounds."""
    errors = []
    if num_cpu <= 0:
        errors.append("FT-004: num_cpu must be > 0")
    if num_gpu < 0:
        errors.append("FT-005: num_gpu must be >= 0")
    if num_memory <= 0:
        errors.append("FT-006: num_memory must be > 0")
    return errors

def validate_training_params(num_epochs: int, learning_rate: float,
                             dataset_fraction: float, train_test_split: float) -> list[str]:
    """FT-008 through FT-011: training parameter ranges."""
    errors = []
    if num_epochs <= 0:
        errors.append("FT-008: num_epochs must be > 0")
    if learning_rate <= 0:
        errors.append("FT-009: learning_rate must be > 0")
    if not (0 < dataset_fraction <= 1):
        errors.append("FT-010: dataset_fraction must be in (0, 1]")
    if not (0 < train_test_split <= 1):
        errors.append("FT-011: train_test_split must be in (0, 1]")
    return errors

def validate_framework_config(framework_type: str, axolotl_config_id: str) -> list[str]:
    """FT-001, FT-012: framework type and config consistency."""
    errors = []
    if framework_type not in ("legacy", "axolotl"):
        errors.append("FT-001: framework_type must be legacy or axolotl")
    if framework_type == "axolotl" and not axolotl_config_id:
        errors.append("FT-012: axolotl_config_id required when framework_type=axolotl")
    return errors
```

## DB-Dependent Validation

Some rules require database lookups. Use `FineTuningStudioClient` for these:

```python
from ft.client import FineTuningStudioClient

client = FineTuningStudioClient()

# Check if model exists (FT-013)
models = client.get_models()
model_ids = {m.id for m in models}
assert model_id in model_ids, f"FT-013: Model {model_id} not found"

# Check if dataset exists (FT-014)
datasets = client.get_datasets()
dataset_ids = {d.id for d in datasets}
assert dataset_id in dataset_ids, f"FT-014: Dataset {dataset_id} not found"

# Check if prompt exists (FT-015)
prompts = client.get_prompts()
prompt_ids = {p.id for p in prompts}
assert prompt_id in prompt_ids, f"FT-015: Prompt {prompt_id} not found"
```

The client connects to the gRPC server over `FINE_TUNING_SERVICE_IP:FINE_TUNING_SERVICE_PORT`. These environment variables are set during Studio initialization.

## Config Content Validation

Validate config content before submitting via `AddConfig`:

```python
import json
import yaml

def validate_config(config_type: str, config_content: str) -> list[str]:
    """CF-002, CF-003: config content must parse correctly."""
    errors = []
    try:
        if config_type == "axolotl":
            yaml.safe_load(config_content)  # CF-003
        else:
            json.loads(config_content)  # CF-002
    except (yaml.YAMLError, json.JSONDecodeError) as e:
        errors.append(f"Config parse error: {e}")
    return errors
```

## Composing a Validation Pipeline

Combine local and DB-dependent validators into a single function that returns all errors at once:

```python
def validate_fine_tuning_request(
    request: StartFineTuningJobRequest,
    client: FineTuningStudioClient,
) -> list[str]:
    """Validate a fine-tuning request against all applicable rules.

    Returns a list of error strings. An empty list means the request is valid.
    """
    errors = []

    # Local validation
    errors.extend(validate_framework_config(
        request.framework_type, request.axolotl_config_id))
    errors.extend(validate_adapter_name(request.adapter_name))
    errors.extend(validate_resource_allocation(
        request.num_cpu, request.num_gpu, request.num_memory))
    errors.extend(validate_training_params(
        request.num_epochs, request.learning_rate,
        request.dataset_fraction, request.train_test_split))

    # DB-dependent validation
    models = {m.id for m in client.get_models()}
    if request.base_model_id not in models:
        errors.append(f"FT-013: Model {request.base_model_id} not found")

    datasets = {d.id for d in client.get_datasets()}
    if request.dataset_id not in datasets:
        errors.append(f"FT-014: Dataset {request.dataset_id} not found")

    if request.framework_type == "legacy":
        prompts = {p.id for p in client.get_prompts()}
        if request.prompt_id not in prompts:
            errors.append(f"FT-015: Prompt {request.prompt_id} not found")

    if request.axolotl_config_id:
        configs = {c.id for c in client.get_configs()}
        if request.axolotl_config_id not in configs:
            errors.append(f"FT-016: Config {request.axolotl_config_id} not found")

    return errors
```

## Usage Pattern

Call the validation pipeline before submitting any request:

```python
from ft.client import FineTuningStudioClient
from ft.proto.fine_tuning_studio_pb2 import StartFineTuningJobRequest

client = FineTuningStudioClient()

request = StartFineTuningJobRequest(
    framework_type="legacy",
    adapter_name="my-adapter",
    base_model_id="abc-123",
    dataset_id="def-456",
    prompt_id="ghi-789",
    num_cpu=2,
    num_gpu=1,
    num_memory=8,
    num_epochs=3,
    learning_rate=2e-5,
    dataset_fraction=1.0,
    train_test_split=0.8,
)

errors = validate_fine_tuning_request(request, client)
if errors:
    for e in errors:
        print(f"  {e}")
    raise ValueError(f"Validation failed with {len(errors)} error(s)")

# Safe to submit
client.start_fine_tuning_job(request)
```

This pattern ensures that invalid requests never reach the gRPC server, providing immediate feedback and avoiding wasted CML Job compute.
