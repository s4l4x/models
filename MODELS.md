# Model Database

Bench environment: RTX 5090 (32GB VRAM), llama.cpp build 2e7e63852 (8173), flash-attn, ngl=99

---

## Qwen3-Coder-30B-A3B-Instruct

**URL:** https://huggingface.co/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF
**Local:** /mnt/data/models/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF/

**Summary:** Coding-specialized MoE model from Alibaba. Optimized for agentic coding, repo-scale
code understanding, and tool/browser automation. Text-only. No thinking mode.

**Features:**
- ✅ Tool calling (OpenAI-compatible)
- ✅ Long context: 256k native, 1M with YaRN
- ✅ Agentic coding / browser automation
- ❌ Vision/multimodal
- ❌ Thinking/reasoning mode

**Architecture:** 30.5B params / 3.3B active | MoE 128 experts, 8 active | 48 layers | 32 Q-heads, 4 KV-heads | head_dim 128

**KV cache @ 131k ctx** (confirmed 48L / 4KVH / 128dim):

| kv type | K mem  | V mem  | total  | model + KV  |
|---------|--------|--------|--------|-------------|
| f16     | 6 GiB  | 6 GiB  | 12 GiB | ~28.5 GiB ✅ |
| q8_0    | 3 GiB  | 3 GiB  | 6 GiB  | ~22.5 GiB ✅ |
| q4_0    | 1.5 GiB| 1.5 GiB| 3 GiB  | ~19.5 GiB ✅ |

### Quant Inventory

| file                                          | quant      | size      | status  |
|-----------------------------------------------|------------|-----------|---------|
| Qwen3-Coder-30B-A3B-Instruct-UD-Q4_K_XL.gguf | UD-Q4_K_XL | 16.45 GiB | ✅ keep |

### Bench Results (2025-02-27)

**Context depth (q8_0 KV):**

| quant      | pp@512   | pp@4096  | pp@16384 | pp@65536 | tg@128  |
|------------|----------|----------|----------|----------|---------|
| UD-Q4_K_XL | 7526 t/s | 7142 t/s | 6188 t/s | 3918 t/s | 273 t/s |

⚠️ Severe long-context pp degradation: pp@65k = 52% of pp@512. Keep context ≤32k for best throughput.

**KV cache type (UD-Q4_K_XL, pp@4096):**

| ctk   | ctv   | pp t/s | tg t/s | total @131k |
|-------|-------|--------|--------|-------------|
| f16   | f16   | 7326   | **287**| ~28.5 GiB ✅ ⭐ |
| q8_0  | q8_0  | 7065   | 269    | ~22.5 GiB ✅ |
| q4_0  | q8_0  | 7146   | 269    | ~22.5 GiB ✅ |
| q4_0  | q4_0  | 7137   | 270    | ~19.5 GiB ✅ |

No meaningful asymmetric KV benefit for this model. f16/f16 fits at 131k and gives best tg (+18 t/s vs q8_0).

**Sweet spot: `f16/f16` at any ctx — best tg, fits comfortably. Fall back to q8_0/q8_0 if sharing GPU.**

### Example Commands

**llama-cli:**
```bash
llama-cli \
  --model /mnt/data/models/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF/Qwen3-Coder-30B-A3B-Instruct-UD-Q4_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 1024 \
  --ubatch-size 1024 \
  --ctx-size 131072 \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --temp 0.7 --min-p 0.0 --top-p 0.8 --top-k 20 --repeat-penalty 1.05 \
  --flash-attn \
  --conversation
```

**llama-server:**
```bash
llama-server \
  --model /mnt/data/models/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF/Qwen3-Coder-30B-A3B-Instruct-UD-Q4_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 1024 \
  --ubatch-size 1024 \
  --ctx-size 131072 \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --temp 0.7 --top-p 0.8 --top-k 20 --repeat-penalty 1.05 \
  --flash-attn \
  --host 0.0.0.0 \
  --port 8080
```

---

## Qwen3.5-35B-A3B

**URL:** https://huggingface.co/unsloth/Qwen3.5-35B-A3B-GGUF
**Local:** /mnt/data/models/unsloth/Qwen3.5-35B-A3B-GGUF/

**Summary:** General-purpose MoE model from Alibaba. Handles text, images, and video with extended
thinking/reasoning mode. Strong at coding, planning, agents, and document understanding.
Newer generation than Qwen3-Coder; use this for non-coding or mixed workloads.

**Features:**
- ✅ Tool calling
- ✅ Vision/multimodal (image, video, documents) — *llama.cpp vision not yet tested*
- ✅ Thinking/reasoning mode (enabled by default, `<think>` blocks)
- ✅ Long context: 262k native, 1M with YaRN
- ✅ Agentic workflows
- ✅ 201 languages

**Architecture:** 35B params / 3B active | MoE 256 experts, 8 routed + 1 shared | layers/KV-heads unconfirmed (est. 64L / 4KVH / 128dim)

**KV cache @ 131k ctx** (estimated — architecture unconfirmed):

| kv type | total (est.) | model + KV      |
|---------|--------------|-----------------|
| f16     | ~16 GiB      | ~35–39 GiB ❌    |
| q8_0    | ~8 GiB       | ~27–31 GiB ✅/⚠️ |
| q4_0    | ~4 GiB       | ~23–27 GiB ✅    |

**Recommended sampling:**
- Thinking mode: `--temp 1.0 --top-p 0.95 --top-k 20 --presence-penalty 1.5`
- Non-thinking: `--temp 0.7 --top-p 0.8 --top-k 20 --presence-penalty 1.5`

### Quant Inventory

| file                              | quant      | size      | status  |
|-----------------------------------|------------|-----------|---------|
| Qwen3.5-35B-A3B-UD-Q4_K_XL.gguf | UD-Q4_K_XL | 19.16 GiB | ✅ keep |
| Qwen3.5-35B-A3B-UD-Q5_K_XL.gguf | UD-Q5_K_XL | 23.21 GiB | ✅ keep |

### Bench Results (2025-02-27)

**Context depth (q8_0 KV):**

| quant      | pp@512   | pp@4096  | pp@16384 | pp@65536 | tg@128  |
|------------|----------|----------|----------|----------|---------|
| UD-Q4_K_XL | 5939 t/s | 5826 t/s | 5669 t/s | 5145 t/s | 176 t/s |
| UD-Q5_K_XL | 5667 t/s | 5703 t/s | 5539 t/s | 4972 t/s | 172 t/s |

Good long-context scaling: pp@65k = 87% of pp@512.

**KV cache type (pp@4096):**

| quant      | ctk   | ctv   | pp t/s | tg t/s | total @131k |
|------------|-------|-------|--------|--------|-------------|
| UD-Q4_K_XL | f16   | f16   | 5850   | 179    | ~35 GiB ❌   |
| UD-Q4_K_XL | q8_0  | q8_0  | 5826   | 176    | ~27 GiB ✅ ⭐ |
| UD-Q4_K_XL | q4_0  | q8_0  | 5882   | **179**| ~25 GiB ✅   |
| UD-Q4_K_XL | q4_0  | q4_0  | 5899   | 177    | ~23 GiB ✅   |
| UD-Q5_K_XL | f16   | f16   | 5747   | 177    | ~39 GiB ❌   |
| UD-Q5_K_XL | q8_0  | q8_0  | 5703   | 172    | ~31 GiB ⚠️  |
| UD-Q5_K_XL | q4_0  | q8_0  | 5767   | **175**| ~29 GiB ✅ ⭐ |
| UD-Q5_K_XL | q4_0  | q4_0  | 5750   | 170    | ~27 GiB ✅   |

`q4_0 K + q8_0 V` is the best asymmetric combo — faster tg AND better quality than q8K+q4V (V cache less sensitive to quantization). +4.7 t/s on Q5, +1.2 t/s on Q4 vs q8K+q4V.

**Sweet spots:**
- `UD-Q4_K_XL + q8_0/q8_0` — 27 GiB, 176 t/s, 5 GiB headroom. Simple daily driver ⭐
- `UD-Q4_K_XL + q4_0 K / q8_0 V` — 25 GiB, 179 t/s, same as f16 tg
- `UD-Q5_K_XL + q4_0 K / q8_0 V` — 29 GiB, 175 t/s. Quality-first, still fits ⭐

### Example Commands

**llama-cli (Q4_K_XL, daily driver):**
```bash
llama-cli \
  --model /mnt/data/models/unsloth/Qwen3.5-35B-A3B-GGUF/Qwen3.5-35B-A3B-UD-Q4_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 1024 \
  --ubatch-size 1024 \
  --ctx-size 131072 \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --temp 0.7 --min-p 0.0 --top-p 0.8 --top-k 20 --repeat-penalty 1.05 \
  --flash-attn \
  --conversation
```

**llama-cli (Q5_K_XL, quality-first):**
```bash
llama-cli \
  --model /mnt/data/models/unsloth/Qwen3.5-35B-A3B-GGUF/Qwen3.5-35B-A3B-UD-Q5_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 1024 \
  --ubatch-size 1024 \
  --ctx-size 131072 \
  --cache-type-k q4_0 \
  --cache-type-v q8_0 \
  --temp 0.7 --min-p 0.0 --top-p 0.8 --top-k 20 --repeat-penalty 1.05 \
  --flash-attn \
  --conversation
```

**llama-server (Q4_K_XL):**
```bash
llama-server \
  --model /mnt/data/models/unsloth/Qwen3.5-35B-A3B-GGUF/Qwen3.5-35B-A3B-UD-Q4_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 1024 \
  --ubatch-size 1024 \
  --ctx-size 131072 \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --temp 0.7 --top-p 0.8 --top-k 20 --repeat-penalty 1.05 \
  --flash-attn \
  --host 0.0.0.0 \
  --port 8080
```
