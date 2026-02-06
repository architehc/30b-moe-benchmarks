# Long Context & Concurrency Guide for 30B MoE Models

**Hardware**: NVIDIA RTX 4090 (24GB VRAM)
**Framework**: llama.cpp
**Models**: 30B MoE models (Nemotron, GLM-4.7-Flash, Qwen3-Coder)

---

## Quick Start

### Maximum Context (768K tokens)
```bash
./build/bin/llama-server \
  -m model.gguf \
  -ngl 99 \
  -c 786432 \
  -np 1 \
  --cache-type-k q4_0 \
  --cache-type-v q4_0 \
  --host 0.0.0.0 \
  --port 8080
```

### Balanced (2 slots × 256K)
```bash
./build/bin/llama-server \
  -m model.gguf \
  -ngl 99 \
  -c 524288 \
  -np 2 \
  --cache-type-k q4_0 \
  --cache-type-v q4_0 \
  --host 0.0.0.0 \
  --port 8080
```

### High Throughput (4 slots × 64K)
```bash
./build/bin/llama-server \
  -m model.gguf \
  -ngl 99 \
  -c 262144 \
  -np 4 \
  --cache-type-k q4_0 \
  --cache-type-v q4_0 \
  --host 0.0.0.0 \
  --port 8080
```

---

## Key Parameters

| Parameter | Description | Recommendation |
|-----------|-------------|----------------|
| `-ngl 99` | GPU layers | Always 99 (full offload) |
| `-c` | Total context size | See configurations below |
| `-np` | Parallel slots | 1-4 depending on use case |
| `--cache-type-k q4_0` | KV cache key quantization | **Required** for long context |
| `--cache-type-v q4_0` | KV cache value quantization | **Required** for long context |

---

## Why KV Cache Quantization is Essential

Without KV cache quantization, a 30B model can only handle ~96K context on 24GB VRAM.

| KV Cache Type | Max Context (24GB) | Memory per 256K |
|---------------|-------------------|-----------------|
| FP16 (default) | ~96K | ~8 GB |
| **Q4_0** | **768K+** | **~864 MB** |

**Always use `--cache-type-k q4_0 --cache-type-v q4_0` for long context workloads.**

---

## Configuration Presets

### 1. Maximum Context (Single User)

Best for: Processing very long documents, code repositories, book-length content

```bash
-c 786432 -np 1 --cache-type-k q4_0 --cache-type-v q4_0
```

| Metric | Value |
|--------|-------|
| Context per slot | 768K tokens |
| Concurrent requests | 1 |
| Prefill speed | ~2,200 tok/s |
| Generation speed | ~50 tok/s |
| KV Cache memory | ~2.6 GB |

### 2. Balanced (Multi-User)

Best for: Multiple users with long context needs

```bash
-c 524288 -np 2 --cache-type-k q4_0 --cache-type-v q4_0
```

| Metric | Value |
|--------|-------|
| Context per slot | 256K tokens |
| Concurrent requests | 2 |
| Aggregate throughput | 5,400-5,700 tok/s |
| Generation speed/req | 70-82 tok/s |

### 3. High Throughput (Many Short Requests)

Best for: API serving, chatbots, quick queries

```bash
-c 262144 -np 4 --cache-type-k q4_0 --cache-type-v q4_0
```

| Metric | Value |
|--------|-------|
| Context per slot | 64K tokens |
| Concurrent requests | 4 |
| Aggregate throughput | 5,400-5,500 tok/s |
| Requests/sec (~3K prompt) | ~1.5 req/s |

### 4. Maximum Concurrency (Short Context)

Best for: High-volume API with short prompts

```bash
-c 131072 -np 8 --cache-type-k q4_0 --cache-type-v q4_0
```

| Metric | Value |
|--------|-------|
| Context per slot | 16K tokens |
| Concurrent requests | 8 |
| Best for | High request volume |

---

## Memory Budget Calculator

### Formula
```
Total VRAM = Model + KV Cache + Compute Buffer

Model (IQ4_XS/Q4_K_M): 16-18 GB
KV Cache (Q4_0): ~3.4 MB per 1K context tokens
Compute Buffer: 1.2-2.3 GB
```

### Examples (24GB VRAM)

| Config | Model | KV Cache | Compute | Total | Fits? |
|--------|-------|----------|---------|-------|-------|
| 1×768K | 17 GB | 2.6 GB | 2.3 GB | 21.9 GB | ✅ |
| 2×256K | 17 GB | 1.7 GB | 2.0 GB | 20.7 GB | ✅ |
| 4×64K | 17 GB | 0.9 GB | 1.5 GB | 19.4 GB | ✅ |
| 1×1M | 17 GB | 3.4 GB | 2.5 GB | 22.9 GB | ⚠️ Tight |

---

## Model-Specific Notes

### Nemotron-30B-A3B (Mamba+MoE)
- **Best for**: Maximum context (768K+ native)
- **Architecture**: Mamba layers use recurrent state instead of KV cache
- **RS Buffer**: Only 48-95 MB regardless of context length
- **Advantage**: Most memory-efficient for very long context

```bash
./build/bin/llama-server \
  -m nvidia_Nemotron-3-Nano-30B-A3B-IQ4_XS.gguf \
  -ngl 99 -c 786432 -np 1 \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --host 0.0.0.0 --port 8080
```

### GLM-4.7-Flash (MoE+MLA)
- **Best for**: Agentic tasks, tool use
- **Note**: Uses MLA (Multi-head Latent Attention) for efficient KV
- **Context**: Supports up to 128K natively

```bash
./build/bin/llama-server \
  -m zai-org_GLM-4.7-Flash-Q4_K_M.gguf \
  -ngl 99 -c 131072 -np 2 \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --host 0.0.0.0 --port 8080
```

### Qwen3-Coder-30B-A3B (MoE)
- **Best for**: Code generation, general tasks
- **Fastest**: 138 tok/s generation
- **Context**: 32K native, extendable

```bash
./build/bin/llama-server \
  -m Qwen3-Coder-30B-A3B-Instruct-IQ4_XS.gguf \
  -ngl 99 -c 65536 -np 2 \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --host 0.0.0.0 --port 8080
```

---

## Benchmarking Your Setup

### Test Throughput
```bash
# Single request throughput
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "test",
    "messages": [{"role": "user", "content": "Write a 500 word essay about AI."}],
    "max_tokens": 512
  }' | jq '.usage'
```

### Test Concurrent Load
```bash
# Install hey: go install github.com/rakyll/hey@latest
hey -n 100 -c 4 -m POST \
  -H "Content-Type: application/json" \
  -d '{"model":"test","messages":[{"role":"user","content":"Hello"}],"max_tokens":50}' \
  http://localhost:8080/v1/chat/completions
```

### Check Memory Usage
```bash
nvidia-smi --query-gpu=memory.used,memory.total --format=csv
```

### Health Check
```bash
curl http://localhost:8080/health
```

---

## Troubleshooting

### Out of Memory
1. Reduce `-c` (context size)
2. Reduce `-np` (parallel slots)
3. Ensure `--cache-type-k q4_0 --cache-type-v q4_0` is set
4. Try a smaller quantization (IQ4_XS vs Q4_K_M)

### Slow Generation
1. Increase `-np` for better batching
2. Check GPU utilization with `nvidia-smi`
3. Ensure full GPU offload with `-ngl 99`

### Context Too Long Error
1. Check total context = slots × context_per_slot
2. Reduce either `-c` or `-np`
3. For Nemotron, can push higher due to Mamba architecture

### Server Won't Start
1. Check if port 8080 is in use: `lsof -i :8080`
2. Kill existing server: `pkill -f llama-server`
3. Check model path is correct
4. Review logs for CUDA errors

---

## Performance Summary

| Configuration | Context | Slots | Throughput | Use Case |
|---------------|---------|-------|------------|----------|
| Max Context | 768K | 1 | 50 tok/s | Long documents |
| Balanced | 256K | 2 | 5,500 tok/s | Multi-user |
| High Throughput | 64K | 4 | 5,500 tok/s | API serving |
| Max Concurrent | 16K | 8 | Variable | High volume |

**Key Insight**: Aggregate throughput plateaus around 5,400-5,700 tok/s regardless of slot count. Choose configuration based on context needs, not raw throughput.

---

## API Usage Example (Python)

```python
import requests

def query_model(prompt, max_tokens=512):
    response = requests.post(
        "http://localhost:8080/v1/chat/completions",
        json={
            "model": "local",
            "messages": [{"role": "user", "content": prompt}],
            "max_tokens": max_tokens,
            "temperature": 0.7
        }
    )
    return response.json()['choices'][0]['message']['content']

# Long context example
with open("large_document.txt") as f:
    document = f.read()

summary = query_model(f"Summarize this document:\n\n{document}", max_tokens=1000)
print(summary)
```

---

*Based on benchmarks run on RTX 4090 with llama.cpp build 7917*
