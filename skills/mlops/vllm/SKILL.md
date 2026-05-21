# Skill: MLOps vLLM Serving Engine

This skill empowers the agent to deploy, configure, scale, and optimize high-throughput LLM inference services using vLLM. It covers launching OpenAI-compatible APIs, configuring GPU tensor parallelism, and managing context lengths.

## System Prompt & Agent Instruction

```text
You are an expert MLOps engineer specialized in high-performance inference and model serving using vLLM.
When the user requests to serve a model or spin up an inference server, you will activate this skill to:
1. Determine the optimal serving parameters (Tensor Parallel size, CPU/GPU swap space, Max Model Len).
2. Launch the vLLM OpenAI-compatible server.
3. Perform system health checks and benchmark throughput.
4. Integrate the server into the gateway configuration.
```

---

## Trigger Conditions

Activate this skill when the conversation involves:
- "Serve model [model_name] on GPU"
- "Start vLLM server"
- "Deploy an OpenAI-compatible API endpoint"
- "Optimize inference speed/throughput using vLLM"
- "Tensor Parallel configuration for multi-GPU serving"

---

## Deployment Configuration Templates

### 1. Single GPU Setup (Low-resource)
```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Meta-Llama-3-8B-Instruct \
    --host 0.0.0.0 \
    --port 8000 \
    --gpu-memory-utilization 0.90 \
    --max-model-len 4096 \
    --quantization awq
```

### 2. Multi-GPU Tensor Parallelism (Production)
```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Meta-Llama-3-70B-Instruct \
    --host 0.0.0.0 \
    --port 8000 \
    --tensor-parallel-size 4 \
    --gpu-memory-utilization 0.95 \
    --max-model-len 8192 \
    --trust-remote-code
```

---

## Step-by-Step Execution Protocol

### Step 1: Resource Auditing
- Execute `nvidia-smi` to find the number of available GPUs and VRAM.
- Map model size to GPU requirements:
  - **7B - 8B Model**: 1x GPU with 16GB-24GB VRAM.
  - **13B - 14B Model**: 1x GPU with 40GB+ VRAM or 2x GPUs (TP=2) with 24GB VRAM.
  - **70B+ Model**: 4x GPUs (TP=4) with 80GB VRAM, or 8x GPUs (TP=8) with 48GB VRAM.

### Step 2: Running vLLM Serving Process
- Start the server using the recommended configuration.
- Capture stdout and stderr to logs (`logs/gateway.log` or a dedicated `logs/vllm.log`).

### Step 3: Health Checking
- Wait for the log pattern: `INFO:     Uvicorn running on http://0.0.0.0:8000`
- Query the model list to verify active serving:
  ```bash
  curl http://localhost:8000/v1/models
  ```
- Run a verification prompt test:
  ```bash
  curl http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "meta-llama/Meta-Llama-3-8B-Instruct",
      "messages": [{"role": "user", "content": "Hello, vLLM!"}],
      "max_tokens": 10
    }'
  ```

---

## Troubleshooting Guide

- **OutOfMemory (OOM) during Warmup / Engine Initialization**:
  1. Reduce `--gpu-memory-utilization` (e.g. from `0.90` to `0.85` or `0.80`).
  2. Reduce `--max-model-len` context window size (e.g. from `8192` to `4096` or `2048`).
  3. Reduce `--max-num-seqs` (default is `256`).
- **NCCL / CUDA Multi-GPU Communications Hang**:
  1. Set environment variable: `export NCCL_DEBUG=INFO`.
  2. Set environment variable: `export NCCL_IB_DISABLE=1` if running inside cloud environments without InfiniBand.
  3. Verify NVLink status via `nvidia-smi topo -m`.
- **Hugging Face Hub Connection Timeout**:
  1. Pre-download the model weights using `huggingface-cli download`.
  2. Launch vLLM referencing the local directory path: `--model /path/to/local/model-dir`.
