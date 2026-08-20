# Qwen 3.8 27B — Best Local Setup Guide

A practical setup guide for running Qwen 3.8 27B locally with optimal settings. Companion to [this video](YOUR_VIDEO_LINK).

## Quick Start

```bash
# 1. Download (Unsloth — has MTP baked in)
pip install huggingface-hub
hf download unsloth/Qwen3.8-27B-GGUF --include "*Q4_K_M*" --local-dir ./models

# 2. Build llama.cpp
git clone --depth 1 https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=native
cmake --build build --config Release -j$(nproc)

# 3. Launch (with MTP + medium reasoning)
./build/bin/llama-server \
  -m ../models/Qwen3.8-27B-Q4_K_M.gguf \
  -c 16384 \
  -ngl 999 \
  --spec-type draft-mtp \
  --spec-draft-n-max 2 \
  --host 0.0.0.0 --port 8080 \
  --chat-template-kwargs '{"reasoning_effort":"medium"}'
```

## Why Unsloth?

Unsloth GGUFs include the MTP (Multi-Token Prediction) draft head. Official Qwen GGUFs don't. MTP gives ~1.5–2x speed on coding tasks with one flag.

## Quant Selection

| VRAM | Recommended Quant | Notes |
|------|-------------------|-------|
| 16GB | IQ4_XS | Fits with small context, MTP may not fit |
| 20GB | Q4_K_M | Fits with 16K context + MTP |
| 24GB | Q6_K | Noticeably cleaner output |
| 32GB+ | Q8_0 | Best quality, room for larger context |

**Rule:** Use the highest quant your VRAM allows. This model is more quant-sensitive than its predecessors.

## Key Flags Explained

| Flag | What it does |
|------|-------------|
| `-ngl 999` | All layers on GPU. Offloading kills speed on dense models |
| `--spec-type draft-mtp` | Enables multi-token prediction (2x speed) |
| `--spec-draft-n-max 2` | MTP predicts 2 tokens ahead per step |
| `-c 16384` | Context size. Increase if VRAM allows |
| `--chat-template-kwargs '{"reasoning_effort":"medium"}'` | Controls thinking depth. Default xhigh burns tokens |

## Reasoning Effort Settings

| Setting | When to use | Trade-off |
|---------|-------------|-----------|
| `low` | Simple tasks, fast responses | Minimal thinking, fastest |
| `medium` | Coding, general work (recommended default) | Balanced thinking + complete output |
| `xhigh` | Complex architecture, deep reasoning | Deep thinking, may consume entire token budget |

### Our test results (same prompt, same hardware):

| | xhigh (default) | medium |
|---|---|---|
| Time | 119s | 64s |
| Tokens | 4,096 | 2,702 |
| Speed | 34.4 tok/s | 42.2 tok/s |
| MTP acceptance | 60% | 79% |
| Output | Empty | Complete function |

## Verify Your Setup

```bash
# Check VRAM usage
nvidia-smi

# Check model metadata
curl -s http://127.0.0.1:8080/v1/models | jq '.data[0].meta | {
  parameters: .n_params,
  model_size_bytes: .size,
  quantization: .ftype,
  runtime_context: .n_ctx,
  training_context: .n_ctx_train
}'

# Quick test
curl -s http://127.0.0.1:8080/v1/chat/completions -d '{
  "model": "qwen",
  "messages": [{"role": "user", "content": "Say hello in one sentence."}],
  "max_tokens": 100
}' | jq '.choices[0].message.content'
```

## VRAM Budget (20GB card)

| Component | Size |
|-----------|------|
| Model weights (Q4_K_M) | ~17 GB |
| MTP draft context | ~1 GB |
| KV cache (16K context) | ~1 GB |
| **Total** | **~19 GB** |

## Limitations

- **Sycophancy:** If you tell it code is broken, it will agree and change it — even when the code is correct. Verify fixes independently.
- **Quant sensitivity:** Q4 works but Q6+ produces cleaner, more efficient output.
- **xhigh default:** Can consume entire token budget on thinking, returning empty output. Start with medium.

## Architecture (Why It Fits on One GPU)

- 64 total layers
- 16 layers: full attention (exact recall)
- 48 layers: Gated DeltaNet (linear attention, constant memory)
- Pattern: 3 DeltaNet → 1 Attention, repeated 16 times
- KV cache: ~16GB instead of ~64GB at full context

## Links

- [Unsloth GGUF (download)](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF)
- [Official Model Card](https://huggingface.co/Qwen/Qwen3.8-27B)
- [llama.cpp](https://github.com/ggml-org/llama.cpp)

## License

Qwen 3.8 27B is released under Apache 2.0.
