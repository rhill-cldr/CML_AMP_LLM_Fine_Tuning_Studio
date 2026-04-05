# Evaluation Job Lifecycle

Evaluation jobs run MLflow evaluation against model+adapter combinations. A single evaluation request can compare multiple adapters against a baseline, with each combination dispatched as a separate CML Job linked by a shared `parent_job_id`.

## Dispatch Architecture

```d2
direction: right
request: Evaluation Request {label: "Evaluation Request\n(N model+adapter combos)"}
dispatch: Dispatch N CML Jobs {label: "Dispatch N CML Jobs\n(shared parent_job_id)"}
mlflow: MLflow Evaluation {label: "MLflow Evaluation\n(metrics + artifacts)"}
compare: Compare Results {label: "Compare Results\n(baseline adapter)"}

request -> dispatch
dispatch -> mlflow
mlflow -> compare
```

A single `StartEvaluationJob` request specifies N model+adapter combinations. The dispatcher fans out into N independent CML Jobs, each running its own MLflow evaluation. All jobs share a `parent_job_id` that groups them for result comparison in the UI.

## Validation

Validation is performed by `ft/evaluation.py::_validate_start_evaluation_job_request()` before any jobs are created.

**Required fields**:

| Field | Rule |
|---|---|
| `model_adapter_combinations` | Non-empty list of model+adapter pairs |
| `dataset_id` | Must exist in DB |
| `prompt_id` | Must exist in DB |
| `cpu` | Valid resource specification |
| `gpu` | Valid resource specification |
| `memory` | Valid resource specification |

**Per-combination validation**:

- Each `base_model_id` in the combinations list must exist in the database.
- Each `adapter_id` must exist in the database, or be an empty string to evaluate the base model without an adapter.
- The referenced dataset and prompt must exist.

## Multi-Adapter Dispatch

For each model+adapter combination in the request, the dispatcher executes the following sequence:

1. **Generate IDs** -- A UUID `job_id` is generated for each individual evaluation run. A shared `parent_job_id` is generated once for the entire batch.
2. **Create directories** -- A result directory and job directory are created for each run.
3. **Find template CML Job** -- The dispatcher locates the `Mlflow_Evaluation_Base_Job` template in the CML project.
4. **Build argument list** -- Each run receives its own argument string containing:

| Argument | Description |
|---|---|
| `base_model_id` | The model to evaluate |
| `adapter_id` | The adapter to apply (empty string for base model only) |
| `dataset_id` | The evaluation dataset |
| `prompt_id` | The prompt template for formatting |
| `result_dir` | Directory for evaluation output |
| `configs` | Evaluation-specific configuration |
| `selected_features` | Dataset features to include |
| `eval_dataset_fraction` | Fraction of dataset to evaluate on |
| `comparison_adapter_id` | The first adapter in the batch, used as the baseline |
| `job_id` | This run's unique identifier |
| `run_number` | Ordinal position in the batch (1-indexed) |

5. **Create CML Job and JobRun** -- A CML Job is created via `cmlapi` with the specified compute resources.
6. **Store EvaluationJob record** -- An `EvaluationJob` record is inserted into the `evaluation_jobs` table with the `parent_job_id` for grouping.

## Parent Job Grouping

All evaluation runs within a batch share the same `parent_job_id`. This enables:

- **UI grouping** -- The Streamlit UI displays evaluation runs grouped by parent, showing all adapter comparisons in a single view.
- **Baseline comparison** -- The first adapter in the `model_adapter_combinations` list is designated as the baseline (`comparison_adapter_id`). All other runs compare their metrics against this baseline.
- **Batch status tracking** -- The overall status of an evaluation batch can be determined by aggregating the statuses of all child jobs sharing the same `parent_job_id`.

## Evaluation Script

The evaluation logic runs inside `ft/scripts/mlflow_evaluation_base_script.py`:

1. **Load model and adapter** -- The base HuggingFace model is loaded, and the optional PEFT adapter is applied via `load_adapted_hf_generation_pipeline()`. This produces a text-generation pipeline.
2. **Load and preprocess dataset** -- The evaluation dataset is loaded, the prompt template is applied to format inputs, and the dataset is sampled to the configured `eval_dataset_fraction`.
3. **Run MLflow evaluation** -- MLflow's evaluation framework is invoked with the configured metrics. Results (metric values and artifacts) are logged to an MLflow experiment.
4. **Log results** -- Evaluation metrics, predictions, and comparison data are persisted in the MLflow tracking store for retrieval by the UI.

## Protobuf Messages

**`StartEvaluationJobRequest`**:

| Field | Description |
|---|---|
| `model_adapter_combinations` | List of model+adapter pairs to evaluate |
| `dataset_id` | Evaluation dataset reference |
| `prompt_id` | Prompt template for input formatting |
| `cpu`, `gpu`, `memory` | Compute resources per evaluation job |
| `configs` | Evaluation configuration (metrics, generation settings) |

**`EvaluationJobMetadata`**:

| Field | Description |
|---|---|
| `id` | Unique evaluation job identifier |
| `type` | Job type identifier |
| `cml_job_id` | CML Job identifier |
| `parent_job_id` | Shared batch identifier |
| `base_model_id` | Evaluated model |
| `dataset_id` | Evaluation dataset |
| `adapter_id` | Applied adapter (empty for base model) |
| `cpu`, `gpu`, `memory` | Allocated resources |
| `configs` | Evaluation configuration |
| `evaluation_dir` | Path to evaluation results |

See [gRPC Service Design](../architecture/grpc-service.md) for the complete evaluation RPC catalog.
