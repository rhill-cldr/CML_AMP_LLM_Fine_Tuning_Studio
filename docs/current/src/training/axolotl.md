# Axolotl Integration

Axolotl is an alternative training framework supported as a first-class `framework_type` alongside the legacy HuggingFace Accelerate + TRL path. It replaces the separate LoRA, BitsAndBytes, and training argument configs with a single YAML configuration file that defines the entire training run.

## Config Structure

Axolotl configurations are stored in the `configs` table with `ConfigType.axolotl`. A template YAML is provided at:

```
ft/config/axolotl/training_config/template.yaml
```

This template defines the baseline Axolotl training configuration. Users can create custom configs by modifying the template values. The YAML file specifies model loading, LoRA parameters, quantization, dataset handling, training hyperparameters, and output settings in a single document.

## Dataset Format Configs

Dataset format definitions are stored as `ConfigType.axolotl_dataset_formats` in the `configs` table. The source files live in:

```
ft/config/axolotl/dataset_formats/
```

Each JSON file defines the expected column structure for a specific Axolotl dataset type (e.g., `alpaca`, `completion`, `sharegpt`). These files are loaded into the database during initialization by:

```
ft/initialize_db.py::InitializeDB.initialize_axolotl_dataset_type_configs()
```

### Pydantic Models

The dataset format structure is defined by two Pydantic models in `ft/api/types.py`:

**`DatasetFormatInfo`**:

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Human-readable name of the dataset format |
| `description` | `str` | The Axolotl dataset type identifier (e.g., `alpaca`, `completion`) |
| `format` | `Dict[str, Any]` | Map of feature column names to their expected types or descriptions |

**`DatasetFormatsCollection`**:

| Field | Type | Description |
|---|---|---|
| `dataset_formats` | `Dict[str, DatasetFormatInfo]` | Map of format names to their definitions |

## Auto-Prompt Generation

When a fine-tuning job uses the axolotl framework and no `prompt_id` is provided, the system automatically generates a prompt template from the dataset format definition. This is handled by `ft/jobs.py::_add_prompt_for_dataset()`.

**Generation steps**:

1. Load the Axolotl YAML config from the database using `axolotl_config_id`.
2. Extract the `type` field from the dataset section of the YAML config. This identifies the expected dataset format (e.g., `alpaca`, `completion`).
3. Query the database for a config of type `axolotl_dataset_formats` whose `description` field matches the extracted type.
4. Parse the dataset format config to extract the feature column names from the `format` dictionary.
5. Generate a default prompt template by concatenating `"Feature: {feature}\n"` for each feature column.
6. Check whether an identical prompt already exists for this dataset to avoid duplicates.
7. Create and return a new prompt record if no duplicate is found.

This mechanism ensures that Axolotl jobs always have a valid prompt template, even when the user does not explicitly create one.

## Legacy vs. Axolotl Comparison

| Aspect | Legacy | Axolotl |
|---|---|---|
| Config format | Separate JSON blobs (LoRA, BnB, training args) | Single YAML file |
| Prompt handling | User must create and select a prompt template | Auto-generated from dataset format if not provided |
| Required configs | `prompt_id` + `lora_config` + `bnb_config` + `training_arguments_config` | `axolotl_config_id` only |
| Training engine | HuggingFace Accelerate + TRL SFTTrainer | Axolotl framework |
| Distributed training | Supported via `dist_*` fields | Managed by Axolotl config |
| Validation | `prompt_id` required | `axolotl_config_id` required; `prompt_id` optional |

## Workflow

To use Axolotl for fine-tuning:

1. Register a base model and dataset via the standard `AddModel` and `AddDataset` RPCs.
2. Create or use an existing Axolotl YAML config (stored as `ConfigType.axolotl`).
3. Call `StartFineTuningJob` with `framework_type = "axolotl"` and `axolotl_config_id` set to the config ID.
4. Omit `prompt_id` to use auto-generation, or provide one to override.
5. The job dispatcher passes `axolotl_config_id` in the `JOB_ARGUMENTS` to the training script, which loads and executes the Axolotl training pipeline.

See [Fine-Tuning Job Lifecycle](./job-lifecycle.md) for the full dispatch flow and [Training Script Architecture](./script-architecture.md) for execution details.
