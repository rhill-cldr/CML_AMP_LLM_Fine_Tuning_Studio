# Model Export & Registry

Trained adapters can be exported through two routes, determined by `ModelExportType`. Both routes merge a base model with a PEFT adapter into a deployable artifact, but target different deployment backends.

## Export Routes

```d2
direction: right
model: Base Model {shape: document}
adapter: PEFT Adapter {shape: document}
pipeline: HF Pipeline {label: "HF Pipeline\n(model + adapter merged)"}
registry: MLflow Registry {shape: cylinder}
cml_model: CML Model Endpoint {shape: cylinder}

model -> pipeline
adapter -> pipeline
pipeline -> registry: "export_model_registry_model()"
pipeline -> cml_model: "deploy_cml_model()"
```

Both routes require non-empty `base_model_id`, `adapter_id`, and `model_name` fields. The choice between them depends on the target deployment environment and the adapter source type.

## MLflow Model Registry

**Function**: `export_model_registry_model()`

This route logs the merged model to the MLflow Model Registry as a registered model. It supports any adapter type (PROJECT, HuggingFace).

**Steps**:

1. **Load pipeline** -- `fetch_pipeline()` creates a HuggingFace text-generation pipeline by loading the base model and applying the PEFT adapter.
2. **Quantized loading** -- If a `BitsAndBytesConfig` is specified, the base model is loaded with 4-bit quantization before adapter application.
3. **Infer signature** -- An MLflow model signature is inferred from example input/output pairs. This defines the expected request and response schema for the registered model.
4. **Log model** -- `mlflow.transformers.log_model()` logs the pipeline to MLflow as a registered model with the specified `model_name`.

**Requirements**:

| Requirement | Detail |
|---|---|
| Base model | HuggingFace model registered in Studio |
| Adapter | Any adapter type (PROJECT or HuggingFace) |
| MLflow tracking | Must be configured in the CML environment |

## CML Model Endpoint

**Function**: `deploy_cml_model()`

This route creates a CML Model endpoint that serves the model+adapter combination as a REST API. It is restricted to PROJECT adapters (file-based, local weights).

**Steps**:

1. **Validate adapter type** -- Only PROJECT adapters (local file-based weights) are supported. HuggingFace adapters must be downloaded locally first.
2. **Create CML Model** -- A CML Model object is created via `cmlapi`.
3. **Create ModelBuild** -- A build is created pointing to the predict script at `ft/scripts/cml_model_predict_script.py`. Environment variables are injected:

| Variable | Description |
|---|---|
| `FINE_TUNING_STUDIO_BASE_MODEL_HF_NAME` | HuggingFace identifier for the base model |
| `ADAPTER_LOCATION` | File path to the adapter weights directory |
| `GEN_CONFIG_STRING` | Serialized generation config (JSON string) |

4. **Deploy** -- A ModelDeployment is created with default resources:

| Resource | Default |
|---|---|
| CPU | 2 cores |
| Memory | 8 GB |
| GPU | 1 |

5. **Resolve runtime** -- The runtime identifier is inherited from the template `Finetuning_Base_Job`, ensuring the model endpoint uses the same environment as training workloads.

**Requirements**:

| Requirement | Detail |
|---|---|
| Base model | HuggingFace model registered in Studio |
| Adapter | PROJECT type only (local file weights) |
| Adapter weights | Must be accessible on the local filesystem |

## Validation

Both export routes perform the following validation before proceeding:

- `base_model_id` must be non-empty and reference an existing model in the database.
- `adapter_id` must be non-empty and reference an existing adapter in the database.
- `model_name` must be non-empty.

Additional route-specific validation:

- **CML Model**: The adapter must be of type PROJECT. Model Registry adapters require the MLflow Registry export path instead.
- **MLflow Registry**: The MLflow tracking server must be accessible.

## Choosing an Export Route

| Criterion | MLflow Registry | CML Model Endpoint |
|---|---|---|
| Adapter source | Any (PROJECT, HuggingFace) | PROJECT only |
| Output format | MLflow registered model | REST API endpoint |
| Serving infrastructure | MLflow serving or downstream consumption | CML Model serving |
| Resource customization | Managed by MLflow | Default 2 CPU / 8 GB / 1 GPU (adjustable post-deploy) |
| Use case | Model versioning, experiment tracking, CI/CD pipelines | Real-time inference endpoint |

See [CML Model Serving](./cml-serving.md) for details on the predict script and endpoint behavior.
