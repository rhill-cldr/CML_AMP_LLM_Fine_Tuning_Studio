# CML Model Serving

A CML Model endpoint serves a fine-tuned model+adapter combination as a REST API. The endpoint is created by `deploy_cml_model()` (see [Model Export & Registry](./model-export.md)) and runs a predict script that loads the model, applies the adapter, and handles inference requests.

## Predict Script

**Path**: `ft/scripts/cml_model_predict_script.py`

The predict script runs inside a CML Model endpoint container. It is specified as the build script during `deploy_cml_model()` and executes in the runtime environment inherited from the template fine-tuning job.

### Initialization

On startup, the script:

1. **Reads environment variables**:

| Variable | Purpose |
|---|---|
| `FINE_TUNING_STUDIO_BASE_MODEL_HF_NAME` | HuggingFace model identifier to load |
| `ADAPTER_LOCATION` | Path to the PEFT adapter weights directory |
| `GEN_CONFIG_STRING` | Serialized generation configuration (JSON) |

2. **Loads the base model** -- The HuggingFace model is loaded from the Hub or cache using the identifier in `FINE_TUNING_STUDIO_BASE_MODEL_HF_NAME`.
3. **Applies the PEFT adapter** -- The LoRA adapter weights at `ADAPTER_LOCATION` are loaded and applied to the base model.

### Request Handling

The predict script exposes a `predict()` function that CML invokes for each incoming request.

**Request format**:

```json
{
  "request": {
    "prompt": "Your input text here"
  }
}
```

The `prompt` field contains the raw input text. The predict function:

1. Extracts the prompt from the request payload.
2. Tokenizes the input using the model's tokenizer.
3. Generates output using the model with the applied generation config.
4. Decodes and returns the generated text.

## Endpoint Creation Flow

The full endpoint creation sequence, initiated by `deploy_cml_model()`:

1. **Create CML Model** -- A new Model object is created in the CML project via `cmlapi`. This registers the model name and description.
2. **Create ModelBuild** -- A build is created with:
   - The predict script path (`ft/scripts/cml_model_predict_script.py`).
   - Environment variables (`FINE_TUNING_STUDIO_BASE_MODEL_HF_NAME`, `ADAPTER_LOCATION`, `GEN_CONFIG_STRING`).
   - The runtime identifier from the template fine-tuning job.
3. **Create ModelDeployment** -- A deployment is created with default resource allocation:

| Resource | Default Value |
|---|---|
| CPU | 2 cores |
| Memory | 8 GB |
| GPU | 1 |

4. **Runtime resolution** -- The runtime is inherited from the template `Finetuning_Base_Job`. This ensures the model endpoint has the same Python packages, CUDA version, and system libraries as the training environment.

## Limitations

- **PROJECT adapters only** -- Only adapters stored as local files (PROJECT type) are supported for CML Model deployment. HuggingFace Hub adapters must be downloaded to the project filesystem before they can be used with a CML Model endpoint.
- **Model Registry adapters** -- Adapters registered through the MLflow Model Registry cannot be deployed as CML Models directly. Use the MLflow Registry export path instead (see [Model Export & Registry](./model-export.md)).
- **Fixed default resources** -- The deployment is created with 1 GPU, 2 CPU cores, and 8 GB memory. To adjust resource allocation after deployment, modify the CML Model settings through the CML UI or `cmlapi`.
- **Single adapter** -- Each CML Model endpoint serves exactly one base model + adapter combination. To serve multiple adapters, create multiple endpoints.

## Post-Deployment

After deployment completes:

- The endpoint URL is available in the CML Model UI and via `cmlapi`.
- Requests are sent as HTTP POST with the JSON format shown above.
- The endpoint auto-scales based on CML's Model serving configuration.
- Logs and metrics are available through CML's standard monitoring interface.
- Resource allocation can be modified via the CML Model settings without rebuilding.
