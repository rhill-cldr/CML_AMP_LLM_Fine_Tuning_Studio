# Fine-Tuning Job Lifecycle

A fine-tuning job trains a PEFT LoRA adapter on a base model using a configured dataset and prompt template. Jobs are dispatched as CML Jobs via the `cmlapi` SDK. The entry point is `ft/jobs.py::start_fine_tuning_job()`, which validates the request, prepares the execution environment, and creates the CML workload.

## Job Dispatch Flow

```d2
direction: right
validate: Validate Request
create_dir: Create Job Directory {label: "Create Job Directory\n(.app/job_runs/{job_id})"}
find_template: Find Template CML Job {label: "Find Template CML Job\n(Accel_Finetuning_Base_Job)"}
build_args: Build Argument List
create_job: Create CML Job + JobRun
store: Store Job Record in DB

validate -> create_dir
create_dir -> find_template
find_template -> build_args
build_args -> create_job
create_job -> store
```

1. **Validate Request** -- `_validate_fine_tuning_request()` checks all fields against the rules below. Any violation raises a `ValueError` that propagates as a gRPC error.
2. **Create Job Directory** -- A UUID `job_id` is generated. The directory `.app/job_runs/{job_id}` is created to hold training artifacts.
3. **Find Template CML Job** -- The dispatcher locates the `Accel_Finetuning_Base_Job` template in the CML project. This template defines the runtime environment and script path.
4. **Build Argument List** -- All training parameters are serialized into a `--key value` string passed as `JOB_ARGUMENTS`.
5. **Create CML Job + JobRun** -- A CML Job and its first JobRun are created via `cmlapi`, with the specified CPU, GPU, and memory resources.
6. **Store Job Record** -- A `FineTuningJob` record is inserted into the `fine_tuning_jobs` table with all metadata for tracking.

## Validation Rules

Validation is performed by `ft/jobs.py::_validate_fine_tuning_request()` before any side effects occur.

| Field | Rule | Error |
|---|---|---|
| `framework_type` | Must be `legacy` or `axolotl` | "framework_type must be either legacy or axolotl" |
| `adapter_name` | Alphanumeric + hyphens only (`^[a-zA-Z0-9-]+$`) | "adapter_name must be alphanumeric" |
| `out_dir` | Must exist as directory | "output_dir does not exist" |
| `num_cpu` | > 0 | "cpu must be greater than 0" |
| `num_gpu` | >= 0 | "gpu must be at least 0" |
| `num_memory` | > 0 | "memory must be greater than 0" |
| `num_workers` | > 0 | "num_workers must be greater than 0" |
| `num_epochs` | > 0 | "Number of epochs must be greater than 0" |
| `learning_rate` | > 0 | "Learning rate must be greater than 0" |
| `dataset_fraction` | (0, 1] | "dataset_fraction must be between 0 and 1" |
| `train_test_split` | (0, 1] | "train_test_split must be between 0 and 1" |
| `axolotl_config_id` | Required when framework=axolotl | "axolotl framework requires axolotl_config_id" |
| `base_model_id` | Must exist in DB | "Model not found" |
| `dataset_id` | Must exist in DB | "Dataset not found" |
| `prompt_id` | Must exist in DB (legacy only) | "Prompt not found" |

## Framework Types

### Legacy

Uses HuggingFace Accelerate with the TRL `SFTTrainer`. The user provides each configuration component separately:

- **prompt_id** -- A prompt template that maps dataset features to the training text format.
- **LoRA config** -- PEFT LoRA hyperparameters (rank, alpha, dropout, target modules).
- **BnB config** -- BitsAndBytes quantization settings (4-bit NF4 quantization).
- **Training arguments** -- Standard HuggingFace `TrainingArguments` fields (epochs, learning rate, batch size, etc.).

For distributed training, worker resources are specified independently via `dist_cpu`, `dist_gpu`, and `dist_mem` fields.

### Axolotl

Uses the Axolotl training framework. The user provides a single YAML configuration file (referenced by `axolotl_config_id`) that bundles all training parameters, LoRA settings, and dataset handling into one document. If no `prompt_id` is provided, the system auto-generates a prompt from the dataset format definition. See [Axolotl Integration](./axolotl.md) for details.

## Resource Specification

Each job requires explicit compute resource allocation:

| Field | Description |
|---|---|
| `num_cpu` | CPU cores for the primary training worker |
| `num_gpu` | GPU count for the primary training worker |
| `num_memory` | Memory in GB for the primary training worker |
| `num_workers` | Number of training workers (Accelerate distributed training) |

For legacy distributed training, additional fields specify per-worker resources:

| Field | Description |
|---|---|
| `dist_cpu` | CPU cores per distributed worker |
| `dist_gpu` | GPU count per distributed worker |
| `dist_mem` | Memory in GB per distributed worker |

## Argument List Schema

Arguments are passed as the `JOB_ARGUMENTS` environment variable to the CML Job. The value is a space-delimited string of `--key value` pairs.

**Core arguments** (always present):

| Key | Source |
|---|---|
| `base_model_id` | Request field |
| `dataset_id` | Request field |
| `experimentid` | Generated UUID (same as `job_id`) |
| `out_dir` | Request field |
| `train_out_dir` | Constructed path for training output |
| `adapter_name` | Request field |
| `framework_type` | Request field (`legacy` or `axolotl`) |

**Optional arguments** (included when non-empty):

| Key | Description |
|---|---|
| `prompt_id` | Prompt template ID (required for legacy, optional for axolotl) |
| `bnb_config` | BitsAndBytes config ID |
| `lora_config` | LoRA config ID |
| `training_arguments_config` | Training arguments config ID |
| `hf_token` | HuggingFace access token |
| `axolotl_config_id` | Axolotl YAML config ID |
| `gpu_label_id` | GPU label config ID |

**Legacy distributed training arguments**:

| Key | Description |
|---|---|
| `dist_num` | Number of distributed workers |
| `dist_cpu` | CPU per worker |
| `dist_mem` | Memory per worker |
| `dist_gpu` | GPU per worker |

## Protobuf Messages

The job lifecycle uses two primary protobuf messages:

- **`StartFineTuningJobRequest`** -- Contains all fields listed above. Sent by the client to initiate training.
- **`StartFineTuningJobResponse`** -- Returns the created job metadata including the generated `job_id` and CML job identifiers.
- **`FineTuningJobMetadata`** -- The full job record stored in the database and returned by `GetFineTuningJob` and `ListFineTuningJobs` RPCs.

See [gRPC Service Design](../architecture/grpc-service.md) for the complete RPC catalog.
