# Introduction

Fine Tuning Studio is a Cloudera AMP (Applied ML Prototype) for managing, fine-tuning, and evaluating large language models within Cloudera Machine Learning (CML). It provides a Streamlit UI backed by a gRPC API, a SQLite metadata store, and job dispatch to CML workloads for training and evaluation. Models, datasets, PEFT adapters, and prompt templates are managed as first-class resources that flow through the import-train-evaluate-deploy lifecycle.

This guide serves two audiences:

| If you are... | Start here |
|---|---|
| **Building a training harness** or extending the platform (custom gRPC clients, new dataset types, training scripts, Axolotl integrations) | [Architecture Reference](./architecture/overview.md) |
| **Building a validation SDK** or CI/CD pipeline for fine-tuning artifacts (config validation, adapter packaging, model export) | [Resource Specifications](./resources/concepts.md) and [Validation Rules](./validation/rules.md) |

## Terminology

| Term | Definition |
|---|---|
| **Dataset** | A reference to a HuggingFace Hub dataset or a local file (CSV, JSON, JSONL) registered in the Studio's metadata store. Features are auto-extracted on import. |
| **Model** | A base foundation model registered from HuggingFace Hub or the CML Model Registry. Serves as the starting point for fine-tuning. |
| **Adapter** | A PEFT LoRA adapter — either produced by a fine-tuning job, imported from a local directory, or fetched from HuggingFace Hub. Applied on top of a base model. |
| **Prompt Template** | A format-string template that maps dataset feature columns into training input. Contains `prompt_template`, `input_template`, and `completion_template` fields. |
| **Config** | A named configuration blob — training arguments, BitsAndBytes quantization, LoRA hyperparameters, generation config, or Axolotl YAML. Configs are deduplicated by content. |
| **Fine-Tuning Job** | A CML Job that trains a PEFT adapter. Dispatched via the gRPC API, tracked in the metadata store, executed as a CML workload with configurable CPU/GPU/memory. |
| **Evaluation Job** | A CML Job that runs MLflow evaluation against one or more model+adapter combinations. Results are tracked in MLflow experiments. |
| **gRPC Service** | The Fine Tuning Service (FTS) — a stateless gRPC server on port 50051 that hosts all application logic. Accessed via `FineTuningStudioClient`. |
| **DAO** | Data Access Object — `FineTuningStudioDao` manages SQLAlchemy sessions and connection pooling against the SQLite database. |
| **CML Workload** | A Cloudera ML Job, session, or model endpoint. Fine-tuning and evaluation are dispatched as CML Jobs via the `cmlapi` SDK. |

## Resource Lifecycle

```d2
direction: right

import: Import Resources {
  shape: document
  label: "Import Resources\n(datasets, models, prompts)"
}
configure: Configure Training {
  label: "Configure Training\n(LoRA, BnB, training args)"
}
train: Fine-Tune {
  label: "Fine-Tune\n(CML Job → PEFT adapter)"
}
evaluate: Evaluate {
  label: "Evaluate\n(MLflow metrics + comparison)"
}
deploy: Export / Deploy {
  label: "Export / Deploy\n(Model Registry or CML Model)"
}

import -> configure
configure -> train
train -> evaluate
evaluate -> deploy
```

The lifecycle begins with importing resources (datasets from HuggingFace or local files, base models, prompt templates) and ends with deploying trained adapters to the CML Model Registry or as CML Model endpoints. The gRPC API drives every step — the Streamlit UI is a client of this API, not the source of truth.
