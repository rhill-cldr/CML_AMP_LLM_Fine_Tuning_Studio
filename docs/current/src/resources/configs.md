# Configuration Specification

A Config resource stores a named configuration blob -- JSON or YAML -- that parameterizes training, quantization, inference, or the Axolotl framework. Configs are content-deduplicated: adding a config with identical content and type to an existing one returns the existing config's ID rather than creating a duplicate.

**Source**: `ft/configs.py`, `ft/consts.py`, `ft/db/model.py`

## Config Types

| Type | Format | Purpose | Default Provided |
|---|---|---|---|
| `training_arguments` | JSON | Training hyperparameters (epochs, optimizer, batch size, learning rate) | Yes |
| `bitsandbytes_config` | JSON | 4-bit quantization settings | Yes |
| `lora_config` | JSON | LoRA hyperparameters | Yes |
| `generation_config` | JSON | Inference generation settings | Yes |
| `custom` | JSON | User-defined configuration blob | No |
| `axolotl` | YAML | Axolotl training configuration file | Template provided |
| `axolotl_dataset_formats` | JSON | Axolotl dataset format schemas | Yes (multiple) |

## ORM Schema

```python
class Config(Base, MappedProtobuf, MappedDict):
    __tablename__ = "configs"
    id = Column(String, primary_key=True)       # UUID
    type = Column(String)                        # ConfigType enum value
    description = Column(String)                 # Model name (for family resolution) or format name
    config = Column(Text)                        # Serialized JSON or YAML string
    model_family = Column(String)                # Architecture family (e.g., "LlamaForCausalLM")
    is_default = Column(Integer, default=1)      # 1 = system/default, 0 = user-created
```

## is_default Semantics

| Value | Constant | Meaning |
|---|---|---|
| `1` | `DEFAULT_CONFIGS` | System-provided default configuration |
| `0` | `USER_CONFIGS` | User-created configuration |

User-created configs always have `is_default=0`. The `add_config()` function sets this automatically.

## Default Config Values

Defined in `ft/consts.py`:

### DEFAULT_TRAINING_ARGUMENTS

```json
{
    "num_train_epochs": 1,
    "optim": "paged_adamw_32bit",
    "per_device_train_batch_size": 1,
    "gradient_accumulation_steps": 4,
    "warmup_ratio": 0.03,
    "max_grad_norm": 0.3,
    "learning_rate": 0.0002,
    "fp16": true,
    "logging_steps": 1,
    "lr_scheduler_type": "constant",
    "disable_tqdm": true,
    "report_to": "mlflow",
    "ddp_find_unused_parameters": false
}
```

### DEFAULT_BNB_CONFIG

```json
{
    "load_in_4bit": true,
    "bnb_4bit_quant_type": "nf4",
    "bnb_4bit_compute_dtype": "float16",
    "bnb_4bit_use_double_quant": true,
    "quant_method": "bitsandbytes"
}
```

### DEFAULT_LORA_CONFIG

```json
{
    "r": 16,
    "lora_alpha": 32,
    "lora_dropout": 0.05,
    "bias": "none",
    "task_type": "CAUSAL_LM"
}
```

### DEFAULT_GENERATIONAL_CONFIG

```json
{
    "do_sample": true,
    "temperature": 0.8,
    "max_new_tokens": 60,
    "top_p": 1,
    "top_k": 50,
    "num_beams": 1,
    "repetition_penalty": 1.1,
    "max_length": null
}
```

## Config Deduplication

`add_config()` implements content-addressed caching:

1. Parse the incoming config string: `yaml.safe_load()` for `axolotl` type, `json.loads()` for all others.
2. Re-serialize to a canonical form (`yaml.dump()` or `json.dumps()`).
3. Query existing configs of the same `type` (and same `model_family` if `description` is provided).
4. Compare parsed content of each existing config against the parsed request content.
5. If an identical config exists, return it. At most one duplicate is expected (asserted).
6. If no match, create a new Config with `is_default=USER_CONFIGS` (0).

When `description` is provided, it is interpreted as a model name: `transform_name_to_family(description)` resolves the HuggingFace architecture (e.g., `"LlamaForCausalLM"`) and scopes the deduplication query to that family.

## Model-Family-Specific Filtering

`list_configs()` applies model-aware filtering when `model_id` is present in the request:

1. Optionally filter by `type` if specified.
2. If `model_id` is provided, call `get_configs_for_model_id()`:
   - Fetch the Model record and resolve `huggingface_model_name`.
   - Instantiate `ModelMetadataFinder(model_hf_name)` and call `fetch_model_family_from_config()`.
   - Filter configs where `model_family` matches and `is_default == 1`.
   - If no model-specific defaults exist, fall back to returning all configs.
3. User configs (`is_default=0`) are not filtered by model family in `get_configs_for_model_id()` -- they are returned when no model-specific defaults are found (fallback behavior).

## Axolotl Config Template

The Axolotl config template is loaded from `ft/config/axolotl/training_config/template.yaml` via `get_axolotl_training_config_template_yaml_str()`. Axolotl dataset format configs are stored in `ft/config/axolotl/dataset_formats/`.

## Protobuf Message

`ConfigMetadata` fields: `id`, `type`, `description`, `config` (serialized JSON/YAML string), `model_family`, `is_default`.
