# Training Script Architecture

The training script is the code that runs inside a CML Job after dispatch. It receives configuration via environment variables, loads and preprocesses data, trains a PEFT LoRA adapter, and saves the result.

## Entry Point

`ft/scripts/accel_fine_tune_base_script.py`

The script is executed as a CML Job. Arguments are received via the `JOB_ARGUMENTS` environment variable as a space-delimited string with `--key value` pairs, parsed into an `argparse` namespace at startup.

## Execution Flow

1. **Parse JOB_ARGUMENTS** -- The `JOB_ARGUMENTS` environment variable is split and parsed via `argparse` into a namespace containing all training parameters.
2. **Load base model** -- The HuggingFace model is loaded with optional `BitsAndBytesConfig` for 4-bit NF4 quantization. The model ID is resolved from the Studio database using `base_model_id`.
3. **Configure tokenizer padding** -- The tokenizer is inspected for a suitable pad token. The function `find_padding_token_candidate()` searches the vocabulary for tokens containing "pad" or "reserved".
4. **Apply PEFT LoRA adapter** -- A `LoraConfig` is constructed from the config blob stored in the database, and the model is wrapped with `get_peft_model()`.
5. **Load and preprocess dataset**:
   - `load_dataset_into_memory()` reads the dataset into a HuggingFace `DatasetDict`.
   - `map_dataset_with_prompt_template()` formats each row using the prompt template, appending the EOS token.
   - `sample_and_split_dataset()` downsamples by the configured fraction and splits into train/test sets (seed=42).
6. **Initialize SFTTrainer** -- A TRL `SFTTrainer` is created with the processed dataset, model, tokenizer, and training arguments.
7. **Train** -- `trainer.train()` executes the training loop.
8. **Save adapter weights** -- The trained LoRA adapter is saved to the output directory.
9. **Auto-register adapter** -- If `auto_add_adapter=true`, the adapter is registered in the Studio database automatically after training completes.

## Dataset Preprocessing Chain

| Step | Function | Input | Output |
|---|---|---|---|
| Load | `load_dataset_into_memory()` | Dataset metadata (type, path, HF name) | HF `DatasetDict` |
| Format | `map_dataset_with_prompt_template()` | `DatasetDict` + prompt template | `DatasetDict` with `prediction` column |
| Sample/Split | `sample_and_split_dataset()` | `DatasetDict` + fraction + split ratio | Train/test `DatasetDict` |

The `prediction` column contains the fully formatted training text for each row -- the prompt template applied to dataset features with the EOS token appended. This column name is defined by `TRAINING_DATA_TEXT_FIELD`.

## Key Training Utilities

All utilities are defined in `ft/training/utils.py`.

### `get_model_parameters(model)`

Returns a tuple of `(total_params, trainable_params)` for the model. Used for logging the parameter count before and after applying the LoRA adapter.

### `map_dataset_with_prompt_template(dataset, template)`

Applies the prompt template to each row in the dataset. The template contains `prompt_template`, `input_template`, and `completion_template` fields that are formatted with the dataset's feature columns. The EOS token is appended to the `prediction` field to signal sequence boundaries during training.

### `sample_and_split_dataset(ds, fraction, split)`

Downsamples the dataset to the specified fraction (e.g., 0.5 = 50% of rows), then splits into train and test sets at the given ratio. Uses `TRAINING_DATASET_SEED = 42` for reproducible splits across runs.

### `find_padding_token_candidate(tokenizer)`

Searches the tokenizer vocabulary for tokens containing "pad" or "reserved" as substrings. Returns the first match found, or `None` if no candidate exists.

### `configure_tokenizer_padding(tokenizer, pad_token)`

Sets the tokenizer's padding token using a fallback chain:

1. Use the tokenizer's existing `pad_token` if already set.
2. Use the provided `pad_token` argument if given.
3. Use the tokenizer's `unk_token` if available.
4. Search for reserved token candidates via `find_padding_token_candidate()`.

This ensures every tokenizer has a valid pad token regardless of the base model's configuration.

## Training Constants

Defined in `ft/consts.py`:

| Constant | Value | Purpose |
|---|---|---|
| `TRAINING_DATA_TEXT_FIELD` | `"prediction"` | Column name for the formatted training text in the preprocessed dataset |
| `TRAINING_DEFAULT_TRAIN_TEST_SPLIT` | `0.9` | Default train/test split ratio (90% train, 10% test) |
| `TRAINING_DEFAULT_DATASET_FRACTION` | `1.0` | Default dataset fraction (use full dataset) |
| `TRAINING_DATASET_SEED` | `42` | Random seed for reproducible dataset splitting and sampling |

## Relationship to Job Lifecycle

The training script is the execution payload created by the [Fine-Tuning Job Lifecycle](./job-lifecycle.md). The job dispatch process builds the `JOB_ARGUMENTS` string, creates the CML Job pointing to this script, and starts a JobRun. The script runs independently inside the CML workload -- it reads its configuration from the environment, accesses the Studio database directly for resource metadata (model paths, dataset locations, config blobs), and writes adapter weights to the output directory.
