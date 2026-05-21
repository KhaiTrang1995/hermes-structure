# Skill: MLOps Axolotl Fine-Tuning

This skill empowers the agent to configure, launch, and monitor LLM fine-tuning jobs using Axolotl. It provides guidelines on selecting datasets, designing YAML configs, and managing GPU resources.

## System Prompt & Agent Instruction

```text
You are an expert MLOps engineer specialized in fine-tuning Large Language Models using Axolotl.
When the user requests to fine-tune a model, or when training tasks are scheduled, you will activate this skill to:
1. Prepare and validate the dataset (JSONL, Hugging Face dataset, etc.).
2. Generate or optimize the Axolotl YAML configuration based on GPU specs and model size.
3. Launch and monitor the training process.
4. Run evaluation benchmarks and manage model checkpoints.
```

---

## Trigger Conditions

Activate this skill when the conversation involves:
- "Fine-tune [model name] on [dataset]"
- "Start training with Axolotl"
- "Create an Axolotl config for Llama-3-8B/Mistral-7B"
- "Debug training loss/OOM error in Axolotl"

---

## Configuration Template (`config.yaml` / YAML block)

The agent should generate or validate the Axolotl training configuration using the template below:

```yaml
base_model: meta-llama/Meta-Llama-3-8B
model_type: LlamaForCausalLM
tokenizer_type: AutoTokenizer

load_in_8bit: false
load_in_4bit: true   # Enable QLoRA for low-memory environments
strict: false

datasets:
  - path: /absolute/path/to/dataset.jsonl
    type: alpaca
    shards: 10
dataset_prepared_path:
val_set_size: 0.05
output_dir: ./outputs/lora-out

adapter: lora
lora_model_dir:

sequence_len: 4096
sample_packing: true
pad_to_sequence_len: true

lora_r: 16
lora_alpha: 32
lora_dropout: 0.05
lora_target_modules:
  - q_proj
  - v_proj
  - k_proj
  - o_proj
  - gate_proj
  - down_proj
  - up_proj

wanderb_project: hermes-axolotl-finetune
wandb_entity:
wandb_watch:
wandb_name:

gradient_accumulation_steps: 4
micro_batch_size: 2
num_epochs: 3
optimizer: adamw_bnb_8bit
lr_scheduler: cosine
learning_rate: 0.0002

train_on_inputs: false
group_by_length: false
bf16: auto
fp16:
tf32: false

gradient_checkpointing: true
early_stopping_patience:
resume_from_checkpoint:
local_rank:
logging_steps: 1
xformers_attention:
flash_attention: true

warmup_steps: 100
evals_per_epoch: 4
eval_table_size:
saves_per_epoch: 1
debug:
deepspeed:
weight_decay: 0.0
fsdp:
fsdp_config:
special_tokens:
  pad_token: "<|end_of_text|>"
```

---

## Step-by-Step Execution Protocol

### Step 1: Pre-flight Verification
- Detect available hardware (`nvidia-smi`) and determine VRAM capacity.
- Choose whether to run Full Fine-Tuning (FFT), LoRA, or QLoRA based on the following scale:
  - **< 24GB VRAM (e.g. RTX 3090/4090)**: Use QLoRA (4-bit quantization), sequence length <= 4096.
  - **24GB - 80GB VRAM (e.g. A10G/A100)**: Use LoRA (16-bit bfloat16), sequence length <= 8192.
  - **Multiple GPUs / > 80GB VRAM**: Use FSDP/DeepSpeed with FFT.

### Step 2: Dataset Verification
- Verify the dataset is properly formatted (`json`, `jsonl`, or HuggingFace identifier).
- Run a lightweight parser check on the dataset paths to avoid runtime parse exceptions.

### Step 3: Run Axolotl Command
Invoke the execution helper script or run directly:
```bash
# Prepare dataset
python -m axolotl.cli.preprocess config.yaml

# Launch training
accelerate launch -m axolotl.cli.train config.yaml
```

---

## Output & Observation Format

During execution, the agent must log periodic training updates in the following format:
```text
[Training Step {step}/{total_steps}] 
- Loss: {current_loss:.4f} (Eval Loss: {eval_loss})
- Learning Rate: {lr}
- GPU VRAM Utilization: {vram_used_gb} GB / {total_vram_gb} GB
- Epoch Progress: {epoch_progress:.2%}
- ETA: {eta}
```

---

## Troubleshooting Guide

- **Out of Memory (OOM) Errors**:
  1. Set `micro_batch_size: 1` and increase `gradient_accumulation_steps` proportionally.
  2. Enable `sample_packing: true`.
  3. Swap `flash_attention: true` or `xformers_attention: true`.
  4. Quantize to 4-bit (`load_in_4bit: true`).
- **Loss Spikes / NaNs**:
  1. Lower the `learning_rate` to `1e-5` or `2e-5`.
  2. Change optimizer to `adamw_torch` or `adamw_apex_fused`.
  3. Check dataset formatting for empty or corrupt inputs.
