# 30B MoE Model Benchmarks for RTX 4090

Comprehensive benchmarks comparing **30B parameter MoE (Mixture of Experts) models** running on a single **NVIDIA RTX 4090 (24GB VRAM)** using llama.cpp.

## Who Is This For?

- **ML Engineers** evaluating which 30B model to deploy for production
- **Hobbyists** wanting to run large models on consumer GPUs
- **Researchers** interested in agentic AI capabilities vs standard benchmarks
- **Developers** building AI-powered coding assistants or agents

## Key Finding

**HumanEval performance does NOT predict agentic capability.**

| Model | HumanEval | SWE-bench | Best For |
|-------|-----------|-----------|----------|
| Qwen3-Coder-30B | **98%** | 0% | Code generation |
| GLM-4.7-Flash | 4% | **20%** | Agentic/SWE tasks |
| Nemotron-30B | 54% | 6.2% | Long context (768K+) |

The model that dominates code generation benchmarks (Qwen3) completely fails at agentic debugging tasks, while GLM excels despite poor HumanEval scores.

## Models Tested

| Model | Quantization | Size | Architecture |
|-------|--------------|------|--------------|
| [Nemotron-30B-A3B](https://huggingface.co/nvidia/Nemotron-3-Nano-30B-A3B) | IQ4_XS | 17 GB | Mamba+MoE |
| [Qwen3-Coder-30B-A3B](https://huggingface.co/Qwen/Qwen3-Coder-30B-A3B) | IQ4_XS | 16 GB | MoE |
| [GLM-4.7-Flash](https://huggingface.co/THUDM/GLM-4-9B-0414) | Q4_K_M | 18 GB | MoE+MLA |

## Benchmarks

### Standard Benchmarks (50 samples each)

| Model | MMLU | GSM8K | HumanEval | Throughput |
|-------|------|-------|-----------|------------|
| Qwen3-Coder | **85%** | **98%** | **98%** | **138 tok/s** |
| Nemotron-30B | 30%* | 82% | 54% | 97 tok/s |
| GLM-4.7-Flash | 30%* | 80% | 4%* | 62 tok/s |

*Output format parsing issues

### SWE-bench Agentic Evaluation (20 instances)

Custom agent loop with bash commands and file editing.

| Model | Patches | Success Rate |
|-------|---------|--------------|
| **GLM-4.7-Flash** | 4/20 | **20%** |
| Nemotron-30B | 2/32 | 6.2% |
| GLM Uncensored | 1/20 | 5% |
| Qwen3-Coder | 0/20 | 0% |

## Documentation

- **[NEMOTRON_BENCHMARK_RESULTS.md](NEMOTRON_BENCHMARK_RESULTS.md)** - Full benchmark results with methodology
- **[LONG_CONTEXT_GUIDE.md](LONG_CONTEXT_GUIDE.md)** - How to run 768K+ context on 24GB VRAM

## Quick Start

### Run GLM-4.7-Flash (Best for Agentic Tasks)
```bash
./llama-server \
  -m GLM-4.7-Flash-Q4_K_M.gguf \
  -ngl 99 -c 131072 -np 2 \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --host 0.0.0.0 --port 8080
```

### Run Qwen3-Coder (Best for Code Generation)
```bash
./llama-server \
  -m Qwen3-Coder-30B-A3B-IQ4_XS.gguf \
  -ngl 99 -c 65536 -np 2 \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --host 0.0.0.0 --port 8080
```

### Run Nemotron (Best for Long Context)
```bash
./llama-server \
  -m Nemotron-30B-A3B-IQ4_XS.gguf \
  -ngl 99 -c 786432 -np 1 \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --host 0.0.0.0 --port 8080
```

## Critical: KV Cache Quantization

**Always use `--cache-type-k q4_0 --cache-type-v q4_0`**

This enables 768K context on 24GB VRAM (vs ~96K without it).

## Recommendations

| Use Case | Model | Why |
|----------|-------|-----|
| **Agentic/SWE tasks** | GLM-4.7-Flash | 20% SWE-bench, best tool-use |
| **Code generation** | Qwen3-Coder | 98% HumanEval, fastest |
| **Math reasoning** | Qwen3-Coder | 98% GSM8K |
| **Long context** | Nemotron-30B | Mamba arch, 768K+ tokens |
| **General knowledge** | Qwen3-Coder | 85% MMLU |

## Hardware

- **GPU**: NVIDIA RTX 4090 (24GB VRAM)
- **Framework**: llama.cpp (build 7917)
- **Quantization**: 4-bit (IQ4_XS, Q4_K_M)

## License

Benchmark results and documentation are released under MIT License.

Model weights are subject to their respective licenses:
- Nemotron: NVIDIA AI License
- Qwen3: Apache 2.0
- GLM: Apache 2.0

---

*Benchmarks conducted February 2026*
