# Qwen 3.8 27B — The Right Way to Run It Locally

![MODEL](https://img.shields.io/badge/MODEL-QWEN%203.8%2027B-blue?style=for-the-badge)
![TESTED ON](https://img.shields.io/badge/🖥%20TESTED%20ON-RTX%20A4500-green?style=for-the-badge)
![RUNTIME](https://img.shields.io/badge/RUNTIME-LLAMA.CPP-orange?style=for-the-badge)
![QUANT](https://img.shields.io/badge/QUANT-Q4__K__M-red?style=for-the-badge)
![LICENSE](https://img.shields.io/badge/LICENSE-APACHE%202.0-purple?style=for-the-badge)
![MTP](https://img.shields.io/badge/MTP-ENABLED-brightgreen?style=for-the-badge)

Run **Qwen 3.8 27B** locally with the best settings for speed, quality, and complete output. Covers architecture, setup, MTP, reasoning effort, and real coding test results.

---

## 🎬 Watch the Video First

[![WATCH THE FULL VIDEO](https://img.shields.io/badge/▶%20WATCH%20THE%20FULL%20VIDEO-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](YOUR_VIDEO_LINK)

### 🧪 Qwen 3.8 27B — Best Local Settings (Complete Guide)

The video contains the complete walkthrough, architecture explanation, 5 real coding tests, reasoning effort comparison, and practical findings.

**This repository is intentionally a companion — not a replacement for the video.** It provides the essential commands and configuration references, while the test prompts, code outputs, and detailed analysis remain in the video.

---

## 📋 What Is Covered?

- ⚡ Setting up Qwen 3.8 27B locally with llama.cpp
- 🧠 Hybrid architecture explained (DeltaNet + Attention)
- 🚀 MTP (Multi-Token Prediction) — how to enable the 2x speed boost
- ⚙️ Reasoning effort settings — why default xhigh can return empty output
- 📊 VRAM budget breakdown for 20GB cards
- 🔍 5 real coding tests with results
- 🛡️ Limitations: sycophancy and quant sensitivity

---

## 🚀 Quick Start

### Step 1: Download the Model

```bash
pip install huggingface-hub
hf download unsloth/Qwen3.8-27B-GGUF --include "*Q4_K_M*" --local-dir ./models
```

> 💡 **Why Unsloth?** Their GGUFs include the MTP draft head baked in. Official Qwen GGUFs don't. One file, no separate draft model needed.

### Step 2: Build llama.cpp

```bash
git clone --depth 1 https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=native
cmake --build build --config Release -j$(nproc)
```

### Step 3: Launch the Server

```bash
./build/bin/llama-server \
  -m ../models/Qwen3.8-27B-Q4_K_M.gguf \
  -c 16384 \
  -ngl 999 \
  --spec-type draft-mtp \
  --spec-draft-n-max 2 \
  --host 0.0.0.0 --port 8080 \
  --chat-template-kwargs '{"reasoning_effort":"medium"}'
```

---

## 🎯 Key Flags Explained

| Flag | What It Does |
|------|-------------|
| `-ngl 999` | Push all layers to GPU. Offloading kills speed on dense models |
| `--spec-type draft-mtp` | Enable multi-token prediction (~2x speed) |
| `--spec-draft-n-max 2` | MTP predicts 2 tokens ahead per step |
| `-c 16384` | Context window size. Increase if VRAM allows |
| `--chat-template-kwargs '{"reasoning_effort":"medium"}'` | Controls thinking depth. See below |

---

## 🧠 Reasoning Effort — The Setting That Changes Everything

| Setting | Best For | Trade-off |
|---------|----------|-----------|
| `low` | Simple tasks, quick answers | Minimal thinking, fastest |
| **`medium`** ⭐ | **Coding, general work (recommended start)** | **Balanced thinking + complete output** |
| `xhigh` | Complex architecture, deep reasoning | Deep thinking, may consume entire token budget |

### 📊 Our Test Results (Same Prompt, Same Hardware)

| | ❌ xhigh (default) | ✅ medium |
|---|---|---|
| ⏱️ Time | 119s | **64s** |
| 🎯 Tokens | 4,096 | **2,702** |
| 🚀 Speed | 34.4 tok/s | **42.2 tok/s** |
| 📈 MTP Acceptance | 60% | **79%** |
| 📄 Output | **Empty** | **Complete function** |

> ⚠️ **xhigh is not broken** — it's designed for genuinely complex tasks. But as a default for everyday coding, medium delivers faster, more complete results. Start with medium, switch to xhigh when the task demands deep reasoning.

---

## 💾 Quant Selection Guide

| VRAM | Recommended Quant | Notes |
|------|-------------------|-------|
| 16GB | `IQ4_XS` | Tight fit. MTP may not fit alongside |
| 20GB | `Q4_K_M` ⭐ | Fits with 16K context + MTP |
| 24GB | `Q6_K` | Noticeably cleaner output |
| 32GB+ | `Q8_0` | Best quality, room for larger context |

> 💡 **This model is more quant-sensitive than its predecessors.** Higher quants produce cleaner output with fewer wasted tokens. Go as high as your VRAM allows.

---

## 📊 VRAM Budget (20GB Card)

| Component | Size |
|-----------|------|
| 🧠 Model weights (Q4_K_M) | ~17 GB |
| ⚡ MTP draft context | ~1 GB |
| 📦 KV cache (16K context) | ~1 GB |
| 💨 Free | ~1 GB |
| **Total** | **~19 / 20 GB (93%)** |

---

## ✅ Verify Your Setup

```bash
# Check VRAM usage after loading
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

---

## 🧪 Test Results Summary

| # | Test | Result |
|---|------|--------|
| 1 | 🔍 Bug Detection (5 planted CSS bugs) | ✅ **5/5 found** with exact fixes — 23 seconds |
| 2 | 🔧 Existing Code Modification | ⚠️ Clean code, matched patterns — **truncated by xhigh** |
| 3 | 📂 Code Quality Review | ⚠️ Good naming, comments, conventions — **with cleanup** |
| 4 | ⚡ Reasoning Effort (xhigh vs medium) | ✅ **Medium: 2x faster + complete output** |
| 5 | 🎯 Sycophancy Trap | ❌ **Agreed with incorrect bug report** |

---

## 🏗️ Architecture — Why It Fits on One GPU

```
64 Total Layers
├── 48 × Gated DeltaNet (Linear Attention) — constant memory
└── 16 × Full Attention (Standard Transformer) — exact recall

Pattern: [DeltaNet] [DeltaNet] [DeltaNet] [Attention] × 16

KV Cache: ~16 GB instead of ~64 GB at full context
Result:  Server-class model → Single GPU
```

---

## ⚠️ Known Limitations

| Limitation | Detail | Mitigation |
|-----------|--------|------------|
| 🤝 Sycophancy | Agrees with incorrect bug reports instead of pushing back | Don't ask "is this wrong?" — describe the behavior, let it diagnose |
| 🎚️ xhigh default | Can burn entire token budget on thinking, returning empty output | Start with `medium`, switch to `xhigh` for complex reasoning |
| 📉 Quant sensitivity | Lower quants produce noticeably worse output than predecessors | Use Q6+ when VRAM allows |

---

## 🔗 Links

| Resource | Link |
|----------|------|
| 📥 Unsloth GGUF (download) | [huggingface.co/unsloth/Qwen3.8-27B-GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF) |
| 📄 Official Model Card | [huggingface.co/Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) |
| 🔨 llama.cpp | [github.com/ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) |

---

## 📜 License

Qwen 3.8 27B is released under **Apache 2.0** — fully free for personal and commercial use.

---

⭐ **Found this useful? Star the repo and share it with anyone running local models.**
