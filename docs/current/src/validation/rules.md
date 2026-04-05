# Validation Rules Reference

The Studio validates resources at multiple points: on import (datasets, models, adapters, prompts, configs), on job submission (fine-tuning, evaluation), and on export (model deployment). This chapter catalogs all validation rules extracted from the source code.

**Source**: `ft/jobs.py`, `ft/evaluation.py`, `ft/datasets.py`, `ft/models.py`, `ft/adapters.py`, `ft/prompts.py`, `ft/configs.py`, `ft/service.py`

## Rule ID Convention

Rule IDs follow the format `{Domain}-{Number}` where Domain is one of:

| Domain | Scope |
|---|---|
| FT | Fine-tuning job parameters |
| EV | Evaluation job parameters |
| DS | Dataset import |
| MD | Model import |
| AD | Adapter import |
| PR | Prompt template |
| CF | Configuration blob |
| EX | Model export / deployment |

All rules with severity `ERROR` abort the operation and return a gRPC error. Rules with severity `INFO` are advisory and do not block the operation.

## Fine-Tuning Job Validation

Validated in `ft/jobs.py` when `StartFineTuningJob` is called.

| Rule ID | Field | Constraint | Severity |
|---|---|---|---|
| FT-001 | `framework_type` | Must be `legacy` or `axolotl` | ERROR |
| FT-002 | `adapter_name` | Must match `^[a-zA-Z0-9-]+$` (alphanumeric + hyphens, no spaces) | ERROR |
| FT-003 | `out_dir` | Must exist as a directory | ERROR |
| FT-004 | `num_cpu` | Must be > 0 | ERROR |
| FT-005 | `num_gpu` | Must be >= 0 | ERROR |
| FT-006 | `num_memory` | Must be > 0 | ERROR |
| FT-007 | `num_workers` | Must be > 0 | ERROR |
| FT-008 | `num_epochs` | Must be > 0 | ERROR |
| FT-009 | `learning_rate` | Must be > 0 | ERROR |
| FT-010 | `dataset_fraction` | Must be in (0, 1] | ERROR |
| FT-011 | `train_test_split` | Must be in (0, 1] | ERROR |
| FT-012 | `axolotl_config_id` | Required when `framework_type=axolotl` | ERROR |
| FT-013 | `base_model_id` | Must exist in models table | ERROR |
| FT-014 | `dataset_id` | Must exist in datasets table | ERROR |
| FT-015 | `prompt_id` | Must exist in prompts table (legacy framework only) | ERROR |
| FT-016 | `axolotl_config_id` | Must exist in configs table (when provided) | ERROR |

FT-001 through FT-011 are local validations that require no database access. FT-012 is a cross-field consistency check. FT-013 through FT-016 are foreign-key validations resolved against the DAO.

## Evaluation Job Validation

Validated in `ft/evaluation.py` when `StartEvaluationJob` is called.

| Rule ID | Field | Constraint | Severity |
|---|---|---|---|
| EV-001 | `model_adapter_combinations` | Must be non-empty | ERROR |
| EV-002 | `dataset_id` | Must be non-empty | ERROR |
| EV-003 | `prompt_id` | Must be non-empty | ERROR |
| EV-004 | `num_cpu`, `num_gpu`, `num_memory` | Must be provided | ERROR |
| EV-005 | model IDs in combinations | Each must exist in models table | ERROR |
| EV-006 | adapter IDs in combinations | Each must exist in adapters table (or empty for base model) | ERROR |
| EV-007 | `dataset_id` | Must exist in datasets table | ERROR |
| EV-008 | `prompt_id` | Must exist in prompts table | ERROR |

Evaluation jobs accept multiple model-adapter pairs in a single request. EV-005 and EV-006 are validated per combination entry.

## Dataset Import Validation

Validated in `ft/datasets.py` when `AddDataset` is called.

| Rule ID | Field | Constraint | Severity |
|---|---|---|---|
| DS-001 | `type` | Must be one of: `huggingface`, `project`, `project_csv`, `project_json`, `project_jsonl` | ERROR |
| DS-002 | `huggingface_name` | Must resolve via `HfApi.dataset_info()` (huggingface type) | ERROR |
| DS-003 | `location` | File must exist (project_csv, project_json, project_jsonl) | ERROR |

DS-002 makes a network call to the HuggingFace Hub. If `HUGGINGFACE_ACCESS_TOKEN` is set, it is used for gated dataset access. DS-003 validates the local filesystem path and checks the file extension matches the declared type.

## Model Import Validation

Validated in `ft/models.py` when `AddModel` is called.

| Rule ID | Field | Constraint | Severity |
|---|---|---|---|
| MD-001 | `type` | Must be one of: `huggingface`, `project`, `model_registry` | ERROR |
| MD-002 | `huggingface_model_name` | Must resolve via `HfApi.model_info()` (huggingface type) | ERROR |
| MD-003 | `cml_registered_model_id` | Must resolve via `cmlapi` (model_registry type) | ERROR |

MD-002 contacts the HuggingFace Hub. MD-003 queries the CML Model Registry through the `cmlapi` SDK.

## Adapter Import Validation

Validated in `ft/adapters.py` when `AddAdapter` is called.

| Rule ID | Field | Constraint | Severity |
|---|---|---|---|
| AD-001 | `name` | Required, non-empty | ERROR |
| AD-002 | `model_id` | Required, must exist in models table | ERROR |
| AD-003 | `location` | Must exist as directory (project type) | ERROR |
| AD-004 | `fine_tuning_job_id` | Must exist in fine_tuning_jobs table (if provided) | ERROR |
| AD-005 | `prompt_id` | Must exist in prompts table (if provided) | ERROR |

AD-004 and AD-005 are optional foreign-key references. When provided, they link the adapter back to the job and prompt that produced it.

## Prompt Validation

Validated in `ft/prompts.py` when `AddPrompt` is called.

| Rule ID | Field | Constraint | Severity |
|---|---|---|---|
| PR-001 | `name` | Required, unique | ERROR |
| PR-002 | `dataset_id` | Required | ERROR |
| PR-003 | `prompt_template` | Required, non-empty | ERROR |
| PR-004 | `input_template` | Required | ERROR |
| PR-005 | `completion_template` | Required | ERROR |

PR-001 enforces uniqueness at the application level before insert. The `prompt_template`, `input_template`, and `completion_template` fields use Python format-string syntax referencing dataset feature column names.

## Config Validation

Validated in `ft/configs.py` when `AddConfig` is called.

| Rule ID | Field | Constraint | Severity |
|---|---|---|---|
| CF-001 | `type` | Must be one of `ConfigType` enum values | ERROR |
| CF-002 | `config` | Must be valid JSON (non-axolotl types) | ERROR |
| CF-003 | `config` | Must be valid YAML (axolotl type) | ERROR |
| CF-004 | `config` | Deduplicated -- returns existing ID if identical content exists | INFO |

CF-004 is a deduplication check, not an error. When a config with identical content already exists, the existing record's ID is returned instead of creating a duplicate. The caller receives a successful response in either case.

## Export Validation

Validated in `ft/models.py` when `ExportModel` or `RegisterModel` is called.

| Rule ID | Field | Constraint | Severity |
|---|---|---|---|
| EX-001 | `base_model_id` | Required, non-empty | ERROR |
| EX-002 | `adapter_id` | Required, non-empty | ERROR |
| EX-003 | `model_name` | Required, non-empty | ERROR |
| EX-004 | adapter type | Must be `PROJECT` for CML Model deployment | ERROR |
| EX-005 | model type | Must be `huggingface` for CML Model deployment | ERROR |

EX-004 and EX-005 enforce deployment constraints. Only project-local adapters (those with files on disk) can be packaged for CML Model Registry, and only HuggingFace-sourced base models are supported for the export merge workflow.
