# Model Database

Bench environment: RTX 5090 (32GB VRAM), llama.cpp build 2e7e63852 (8173), flash-attn, ngl=99

---

## Qwen3-Coder-30B-A3B-Instruct

**URL:** https://huggingface.co/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF
**Released:** 2025-07-31 | **Updated:** 2026-01-30 | **Likes:** 508
**Local:** /mnt/data/models/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF/

**Summary:** Coding-specialized MoE model from Alibaba. Optimized for agentic coding, repo-scale
code understanding, and tool/browser automation. Text-only. No thinking mode.

**Features:**
- ✅ Tool calling (OpenAI-compatible)
- ✅ Long context: 256k native, 1M with YaRN
- ✅ Agentic coding / browser automation
- ❌ Vision/multimodal
- ❌ Thinking/reasoning mode

**Architecture:** 30.5B params / 3.3B active | MoE 128 experts, 8 active | 48 layers | 32 Q-heads, 4 KV-heads | head_dim 128 — see bench results for VRAM usage.

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
  --flash-attn 1 \
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
  --flash-attn 1 \
  --host 0.0.0.0 \
  --port 8080
```

---

## Qwen3.5-35B-A3B

**URL:** https://huggingface.co/unsloth/Qwen3.5-35B-A3B-GGUF
**Released:** 2026-02-24 | **Updated:** 2026-02-27 | **Likes:** 309
**Local:** /mnt/data/models/unsloth/Qwen3.5-35B-A3B-GGUF/

**Summary:** General-purpose MoE model from Alibaba. Handles text, images, and video with extended thinking/reasoning mode. Strong at coding, planning, agents, and document understanding. Newer generation than Qwen3-Coder; use this for non-coding or mixed workloads.

**Features:**
- ✅ Tool calling
- ✅ Vision/multimodal (image, video, documents) — mmproj: `mmproj-F16.gguf`
- ✅ Thinking/reasoning mode (enabled by default, `<think>` blocks)
- ✅ Long context: 262k native, 1M with YaRN
- ✅ Agentic workflows
- ✅ 201 languages

**Architecture:** 35B params / 3B active | MoE 256 experts, 8 routed + 1 shared | layers/KV-heads unconfirmed (est. 64L / 4KVH / 128dim) — see bench results for VRAM usage.

**Recommended sampling:**

| Mode | Use case | temp | top-p | top-k | presence-penalty |
|------|----------|------|-------|-------|-----------------|
| Thinking | General tasks | 1.0 | 0.95 | 20 | 1.5 |
| Thinking | Precise coding | 0.6 | 0.95 | 20 | 0.0 |
| Non-thinking | General tasks | 0.7 | 0.8 | 20 | 1.5 |
| Non-thinking | Reasoning tasks | 1.0 | 0.95 | 20 | 1.5 |

Use `--presence-penalty` (not `--repeat-penalty`) as the anti-repetition knob for this model.

**To disable thinking mode** (non-thinking/instruct behavior):
```
--chat-template-kwargs '{"enable_thinking": false}'
```

### Quant Inventory

| file                              | quant      | size      | status  |
|-----------------------------------|------------|-----------|---------|
| Qwen3.5-35B-A3B-UD-Q4_K_XL.gguf | UD-Q4_K_XL | 19.16 GiB | ✅ keep |
| Qwen3.5-35B-A3B-UD-Q5_K_XL.gguf | UD-Q5_K_XL | 23.21 GiB | ✅ keep |
| mmproj-F16.gguf                   | F16        | 0.9 GiB   | ✅ keep (required for vision) |

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
  --flash-attn 1 \
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
  --flash-attn 1 \
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
  --flash-attn 1 \
  --host 0.0.0.0 \
  --port 8080
```

### Vision Usage

> **Note:** Vision requires `llama-mtmd-cli`, NOT `llama-cli`. The standard `llama-cli` does not
> support `--mmproj` or `--image`. `llama-server` supports `--mmproj` for API-based vision.

#### llama-mtmd-cli (interactive conversation with images)
```bash
llama-mtmd-cli \
  --model /mnt/data/models/unsloth/Qwen3.5-35B-A3B-GGUF/Qwen3.5-35B-A3B-UD-Q4_K_XL.gguf \
  --mmproj /mnt/data/models/unsloth/Qwen3.5-35B-A3B-GGUF/mmproj-F16.gguf \
  --n-gpu-layers 99 \
  --ctx-size 16384 \
  --cache-type-k q8_0 --cache-type-v q8_0 \
  --flash-attn 1 \
  --temp 0.7 --top-p 0.8 --top-k 20
```
> **Note:** Do NOT use `--jinja` with `llama-mtmd-cli` — it conflicts with the vision template handling and causes a crash (`std::out_of_range`). Use `--jinja` only for text-only inference with `llama-cli`/`llama-server`.
Then type your prompt. To include an image in conversation: `/path/to/image.jpg\nYour prompt here`

#### llama-server (API-based vision)
```bash
llama-server \
  --model /mnt/data/models/unsloth/Qwen3.5-35B-A3B-GGUF/Qwen3.5-35B-A3B-UD-Q4_K_XL.gguf \
  --mmproj /mnt/data/models/unsloth/Qwen3.5-35B-A3B-GGUF/mmproj-F16.gguf \
  --n-gpu-layers 99 \
  --ctx-size 16384 \
  --cache-type-k q8_0 --cache-type-v q8_0 \
  --flash-attn 1 \
  --host 0.0.0.0 --port 8080
```

#### API — image from URL
```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{
      "role": "user",
      "content": [
        {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}},
        {"type": "text", "text": "What do you see in this image?"}
      ]
    }],
    "max_tokens": 512
  }'
```

#### API — image from local file (base64)
```bash
IMG_B64=$(base64 -w0 /path/to/image.jpg)
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/jpeg;base64,\${IMG_B64}\"}},
        {\"type\": \"text\", \"text\": \"Describe this image.\"}
      ]
    }],
    \"max_tokens\": 512
  }"
```

---

## Gemma-3-27B-IT

**URL:** https://huggingface.co/unsloth/gemma-3-27b-it-GGUF
**Released:** 2025-03-12 | **Updated:** 2025-08-14 | **Likes:** 188
**Local:** /mnt/data/models/unsloth/gemma-3-27b-it-GGUF/

**Summary:** Google's Gemma 3 27B instruction-tuned vision-language model. Strong multimodal
capabilities with a SigLIP vision encoder. Good for image understanding, OCR, document analysis.
140+ languages.

**Features:**
- ✅ Vision/multimodal (images → text, SigLIP encoder, 896×896 input, 256 tokens/image)
- ✅ Tool calling
- ✅ Long context: 128k native
- ✅ 140+ languages
- ❌ Thinking/reasoning mode

**Architecture:** 27B params | Dense | 62 layers | 32 Q-heads, **16 KV-heads** | head_dim 128
**Hybrid attention:** sliding_window=1024, pattern=6 — only ~10 global layers keep full context KV.
Other 52 layers cap at 1024 tokens in KV cache.

**KV cache @ 131k ctx** (hybrid sliding window — much cheaper than naive calculation):

| kv type | global layers (10) | local layers (52) | total KV | model + KV  |
|---------|--------------------|-------------------|----------|-------------|
| f16     | ~10 GiB            | ~0.4 GiB          | ~10 GiB  | ~26 GiB ✅  |
| q8_0    | ~5 GiB             | ~0.2 GiB          | ~5 GiB   | ~21 GiB ✅  |
| q4_0    | ~2.5 GiB           | ~0.1 GiB          | ~2.6 GiB | ~19 GiB ✅  |

### Quant Inventory

| file                              | quant      | size      | status  |
|-----------------------------------|------------|-----------|---------|
| gemma-3-27b-it-UD-Q4_K_XL.gguf  | UD-Q4_K_XL | 16 GiB    | ✅ keep |
| mmproj-F32.gguf                   | F32        | 1.6 GiB   | ✅ keep (required for vision) |

### Vision Usage

> **Note:** Vision requires `llama-mtmd-cli`, NOT `llama-cli`. The standard `llama-cli` does not
> support `--mmproj` or `--image`. `llama-server` supports `--mmproj` for API-based vision.

#### llama-mtmd-cli (interactive conversation with images)
```bash
llama-mtmd-cli \
  --model /mnt/data/models/unsloth/gemma-3-27b-it-GGUF/gemma-3-27b-it-UD-Q4_K_XL.gguf \
  --mmproj /mnt/data/models/unsloth/gemma-3-27b-it-GGUF/mmproj-F32.gguf \
  --ctx-size 16384 \
  --n-gpu-layers 99 \
  --temp 1.0 --repeat-penalty 1.0 --min-p 0.01 --top-k 64 --top-p 0.95 \
  --seed 3407 --prio 2
```
Then type your prompt. To include an image in conversation: `/path/to/image.jpg\nYour prompt here`

#### llama-server (API-based vision)
```bash
llama-server \
  --model /mnt/data/models/unsloth/gemma-3-27b-it-GGUF/gemma-3-27b-it-UD-Q4_K_XL.gguf \
  --mmproj /mnt/data/models/unsloth/gemma-3-27b-it-GGUF/mmproj-F32.gguf \
  --n-gpu-layers 99 --flash-attn 1 \
  --ctx-size 16384 \
  --cache-type-k q8_0 --cache-type-v q8_0 \
  --host 0.0.0.0 --port 8080
```

#### API — image from URL
```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{
      "role": "user",
      "content": [
        {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}},
        {"type": "text", "text": "What do you see in this image?"}
      ]
    }],
    "max_tokens": 512
  }'
```

#### API — image from local file (base64)
```bash
IMG_B64=$(base64 -w0 /path/to/image.jpg)
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/jpeg;base64,${IMG_B64}\"}},
        {\"type\": \"text\", \"text\": \"Describe this image.\"}
      ]
    }],
    \"max_tokens\": 512
  }"
```

---

## Qwen3-4B-Instruct-2507

**URL:** https://huggingface.co/unsloth/Qwen3-4B-Instruct-2507-GGUF
**Released:** 2025-08-06 | **Updated:** 2025-08-20 | **Likes:** 156
**Local:** /mnt/data/models/unsloth/Qwen3-4B-Instruct-2507-GGUF/

**Summary:** Compact 4B instruction model from Alibaba. Fast, capable for quick tasks, tool
calling, and routing. No thinking mode — direct response only.

**Features:**
- ✅ Tool calling (OpenAI-compatible)
- ✅ Long context: 262k native
- ❌ Vision/multimodal
- ❌ Thinking/reasoning mode

**Architecture:** 4B params | Dense | 36 layers | 32 Q-heads, 8 KV-heads | head_dim 128 — see bench results for VRAM usage.

### Quant Inventory

| file                                  | quant      | size     | status  |
|---------------------------------------|------------|----------|---------|
| Qwen3-4B-Instruct-2507-UD-Q4_K_XL.gguf | UD-Q4_K_XL | 2.4 GiB | ✅ keep |
| ~~Qwen3-4B-Instruct-2507-Q8_0.gguf~~   | Q8_0       | 4.0 GiB  | ❌ deleted — slower tg than Q4_K_XL (see bench) |

### Bench Results (2026-02-27)

Instruct and Thinking models have identical benchmark performance (same architecture, same weights structure).
Results below apply to both.

**Context depth (q8_0 KV):**

| quant      | pp@512    | pp@4096   | pp@16384  | pp@65536 | tg@128      |
|------------|-----------|-----------|-----------|----------|-------------|
| UD-Q4_K_XL | 19,655 t/s | 17,859 t/s | 13,572 t/s | 6,153 t/s | **311 t/s** |
| Q8_0       | 20,509 t/s | 18,391 t/s | 13,906 t/s | 6,207 t/s | 238 t/s     |

⚠️ **Q8_0 is slower at tg than Q4_K_XL (+30% faster on Q4)** — tg is memory-bandwidth-bound.
Smaller model = faster weight reads per token. Q8_0 is only ~5% faster at pp. Delete Q8_0.

**KV cache type (UD-Q4_K_XL, pp@4096):**

| ctk   | ctv   | pp t/s | tg t/s      | total @131k |
|-------|-------|--------|-------------|-------------|
| f16   | f16   | 18,784 | **331** ⭐  | ~7 GiB ✅   |
| q8_0  | q8_0  | 18,131 | 310         | ~4.7 GiB ✅ |
| q4_0  | q8_0  | 18,081 | 313         | ~3.5 GiB ✅ |
| q4_0  | q4_0  | 18,325 | 312         | ~3.5 GiB ✅ |

f16 KV is trivially small for this model (~4.5 GiB total), no reason to use anything else.

**Sweet spot: `UD-Q4_K_XL + f16/f16` — 331 t/s tg, ~7 GiB total. Fits anywhere.**

### Example Commands

**llama-cli:**
```bash
llama-cli \
  --model /mnt/data/models/unsloth/Qwen3-4B-Instruct-2507-GGUF/Qwen3-4B-Instruct-2507-UD-Q4_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 2048 \
  --ubatch-size 512 \
  --ctx-size 32768 \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --temp 0.7 --min-p 0.0 --top-p 0.8 --top-k 20 --repeat-penalty 1.05 \
  --flash-attn 1 \
  --conversation
```

**llama-server:**
```bash
llama-server \
  --model /mnt/data/models/unsloth/Qwen3-4B-Instruct-2507-GGUF/Qwen3-4B-Instruct-2507-UD-Q4_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 2048 \
  --ubatch-size 512 \
  --ctx-size 32768 \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --temp 0.7 --top-p 0.8 --top-k 20 --repeat-penalty 1.05 \
  --flash-attn 1 \
  --host 0.0.0.0 --port 8080
```

---

## Qwen3-4B-Thinking-2507

**URL:** https://huggingface.co/unsloth/Qwen3-4B-Thinking-2507-GGUF
**Released:** 2025-08-06 | **Updated:** 2025-09-11 | **Likes:** 94
**Local:** /mnt/data/models/unsloth/Qwen3-4B-Thinking-2507-GGUF/

**Summary:** Compact 4B thinking/reasoning model from Alibaba. Always-on thinking mode —
generates internal reasoning before answering. Same architecture as Instruct variant but
tuned for extended reasoning chains. Surprisingly capable for its size (AIME25: 81.3).

**Features:**
- ✅ Thinking/reasoning mode (always on, cannot be disabled)
- ✅ Long context: 262k native
- ❌ Vision/multimodal
- ❌ Tool calling (limited — thinking mode conflicts)

**Architecture:** 4B params | Dense | 36 layers | 32 Q-heads, 8 KV-heads | head_dim 128 — see Instruct bench results for VRAM usage.

**Multi-turn note:** Do NOT include `<think>` content in conversation history — only pass the final answer to the model on subsequent turns.

**Output length:** 32,768 tokens for standard queries; up to 81,920 for complex math/programming. Use `--ctx-size 131072` or higher for reasoning tasks.

### Quant Inventory

| file                                 | quant      | size     | status  |
|--------------------------------------|------------|----------|---------|
| Qwen3-4B-Thinking-2507-UD-Q4_K_XL.gguf | UD-Q4_K_XL | 2.4 GiB | ✅ keep |
| ~~Qwen3-4B-Thinking-2507-Q8_0.gguf~~   | Q8_0       | 4.0 GiB  | ❌ deleted — see Instruct bench |

### Bench Results (2026-02-27)

See Qwen3-4B-Instruct-2507 bench results — identical performance (same architecture).
Sweet spot: `UD-Q4_K_XL + f16/f16` — 331 t/s tg, ~7 GiB total.

### Example Commands

**llama-cli:**
```bash
llama-cli \
  --model /mnt/data/models/unsloth/Qwen3-4B-Thinking-2507-GGUF/Qwen3-4B-Thinking-2507-UD-Q4_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 2048 \
  --ubatch-size 512 \
  --ctx-size 131072 \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --temp 0.6 --min-p 0.0 --top-p 0.95 --top-k 20 \
  --flash-attn 1 \
  --conversation
```

**llama-server:**
```bash
llama-server \
  --model /mnt/data/models/unsloth/Qwen3-4B-Thinking-2507-GGUF/Qwen3-4B-Thinking-2507-UD-Q4_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 2048 \
  --ubatch-size 512 \
  --ctx-size 131072 \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --temp 0.6 --top-p 0.95 --top-k 20 \
  --flash-attn 1 \
  --host 0.0.0.0 --port 8080
```

---

## gpt-oss-20b

**URL:** https://huggingface.co/unsloth/gpt-oss-20b-GGUF
**Released:** 2025-08-05 | **Updated:** 2025-12-19 | **Likes:** 610
**Local:** /mnt/data/models/unsloth/gpt-oss-20b-GGUF/

**Summary:** OpenAI's open-weight MoE model designed for lower latency and local/specialized use cases. Supports configurable reasoning levels (low/medium/high), tool use, and structured outputs. Licensed Apache 2.0.

**Features:**
- ✅ Tool calling (function calling, structured outputs)
- ❌ Vision/multimodal
- ✅ Reasoning levels: low / medium / high (configurable via system prompt)
- ✅ Context: 131k (YaRN from 4096 native)

**Architecture:** 21B total / 3.6B active | MoE (32 experts, 4 active) | 24L / 8KVH / 64dim (confirmed) | hybrid sliding window (12 local w=128, 12 full-attention)

**Note:** Trained on OpenAI's Harmony response format — chat template embedded in tokenizer handles this automatically in llama.cpp.

**Reasoning level** — set via system prompt (include as first line or standalone system message):
```
Reasoning: low     # fast, minimal chain-of-thought
Reasoning: medium  # balanced (default behavior)
Reasoning: high    # deep analysis, longer think time
```

**KV cache (full-attention layers only — 12/24 contribute to long context):**

| ctx   | f16   | q8_0  | q4_0  |
|-------|-------|-------|-------|
| 8k    | ~0.2G | ~0.1G | ~0.1G |
| 32k   | ~0.7G | ~0.4G | ~0.2G |
| 131k  | ~3.0G | ~1.5G | ~0.8G |

f16 KV trivially fits at any context (11G model + 3G KV = 14G total at 131k).

### Quant Inventory

| file | quant | size | status |
|------|-------|------|--------|
| gpt-oss-20b-Q4_K_S.gguf | Q4_K_S | 11 GiB | ✅ keep |

### Bench Results (2026-02-27)

**Context depth (f16 KV):**

| quant   | pp@512     | pp@4096    | pp@16384   | pp@65536  | tg@128      |
|---------|------------|------------|------------|-----------|-------------|
| Q4_K_S  | 13414 t/s  | 13745 t/s  | 12810 t/s  | 9738 t/s  | **368 t/s** |

Moderate long-context degradation: pp@65k = 73% of pp@512.

**KV cache type (Q4_K_S, pp@4096):**

| ctk   | ctv   | pp t/s | tg t/s     | total @131k |
|-------|-------|--------|------------|-------------|
| f16   | f16   | 13745  | **368** ⭐ | ~14 GiB ✅  |
| q8_0  | q8_0  | 13657  | 369        | ~12.5 GiB ✅ |
| q4_0  | q8_0  | 13791  | 359        | ~11.8 GiB ✅ |
| q4_0  | q4_0  | 13729  | 365        | ~11.4 GiB ✅ |

KV cache is trivially small (~3 GiB f16 at 131k) — no reason to quantize it. All configs within noise; f16/f16 is simplest and tied-best.

**Sweet spot: `Q4_K_S + f16/f16` — 368 t/s tg, ~14 GiB total. Use f16 KV at any context.**

### Example Commands

**llama-cli:**
```bash
llama-cli \
  --model /mnt/data/models/unsloth/gpt-oss-20b-GGUF/gpt-oss-20b-Q4_K_S.gguf \
  --n-gpu-layers 99 \
  --batch-size 2048 \
  --ubatch-size 512 \
  --ctx-size 65536 \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --temp 0.6 --top-p 0.95 --top-k 20 \
  --flash-attn 1 \
  --conversation
```

**llama-server:**
```bash
llama-server \
  --model /mnt/data/models/unsloth/gpt-oss-20b-GGUF/gpt-oss-20b-Q4_K_S.gguf \
  --n-gpu-layers 99 \
  --batch-size 2048 \
  --ubatch-size 512 \
  --ctx-size 65536 \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --temp 0.6 --top-p 0.95 --top-k 20 \
  --flash-attn 1 \
  --host 0.0.0.0 --port 8080
```

---

## Qwen3.5-27B

**URL:** https://huggingface.co/unsloth/Qwen3.5-27B-GGUF
**Released:** 2026-02-24 | **Updated:** 2026-03-02 | **Likes:** 211
**Local:** /mnt/data/models/unsloth/Qwen3.5-27B-GGUF/

**Summary:** Alibaba's unified vision-language foundation model with a hybrid Gated DeltaNet + Attention + MoE architecture. Early fusion multimodal training, extended thinking, tool use, 201 languages. Same generation as Qwen3.5-35B-A3B but dense-ish 27B with a novel linear-attention hybrid.

**Features:**
- ✅ Tool calling (OpenAI-compatible)
- ✅ Vision/multimodal (image, video, documents) — mmproj: `mmproj-F16.gguf`
- ✅ Thinking/reasoning mode (enabled by default, `<think>` blocks)
- ✅ Long context: 262k native, 1M with YaRN
- ✅ 201 languages

**Architecture:** 27B params | Hybrid DeltaNet + Attention + sparse MoE | 64 layers
Hidden layout: 16 × (3 × (Gated DeltaNet → FFN) → 1 × (Gated Attention → FFN))
- **DeltaNet layers (48):** 48 V-heads / 16 QK-heads, head_dim 128 (linear attention — no KV cache)
- **Attention layers (16):** 24 Q-heads / 4 KV-heads, head_dim 256, RoPE dim 64
- FFN intermediate: 17,408, sparse MoE
- Only 16/64 layers maintain KV cache → very efficient at long context

**KV cache estimate (16 attention layers, 4 KVH, head_dim 256):**

| ctx   | f16    | q8_0   | q4_0   |
|-------|--------|--------|--------|
| 32k   | ~2.1G  | ~1.1G  | ~0.5G  |
| 131k  | ~8.6G  | ~4.3G  | ~2.1G  |

**Recommended sampling:**

| Mode | Use case | temp | top-p | top-k | presence-penalty |
|------|----------|------|-------|-------|-----------------|
| Thinking | General tasks | 1.0 | 0.95 | 20 | 1.5 |
| Thinking | Precise coding | 0.6 | 0.95 | 20 | 0.0 |
| Non-thinking | General tasks | 0.7 | 0.8 | 20 | 1.5 |
| Non-thinking | Reasoning tasks | 1.0 | 1.0 | 40 | 2.0 |

Use `--presence-penalty` (not `--repeat-penalty`) as the anti-repetition knob for this model.

**To disable thinking mode** (non-thinking/instruct behavior):
```
--chat-template-kwargs '{"enable_thinking": false}'
```

### Quant Inventory

| file | quant | size | status |
|------|-------|------|--------|
| Qwen3.5-27B-UD-Q4_K_XL.gguf | UD-Q4_K_XL | 17.6 GiB | ✅ keep |
| Qwen3.5-27B-UD-Q5_K_XL.gguf | UD-Q5_K_XL | 20.2 GiB | ✅ keep |
| mmproj-F16.gguf | F16 | 0.9 GiB | ✅ keep (required for vision) |

### Bench Results (2026-03-04)

**Context depth (q8_0 KV):**

| quant      | pp@512   | pp@4096  | pp@16384 | pp@65536 | tg@128 |
|------------|----------|----------|----------|----------|--------|
| UD-Q4_K_XL | 3237 t/s | 3195 t/s | 3116 t/s | 2399 t/s | 61 t/s |
| UD-Q5_K_XL | 3117 t/s | 3072 t/s | 2907 t/s | 2302 t/s | 55 t/s |

Moderate long-context degradation: pp@65k = 74% of pp@512.

**KV cache type (pp@4096):**

| quant      | ctk   | ctv   | pp t/s | tg t/s | total @131k |
|------------|-------|-------|--------|--------|-------------|
| UD-Q4_K_XL | f16   | f16   | 3264   | **62** | ~26 GiB ✅ ⭐ |
| UD-Q4_K_XL | q8_0  | q8_0  | 3195   | 61     | ~22 GiB ✅  |
| UD-Q4_K_XL | q4_0  | q8_0  | 3279   | 60     | ~20 GiB ✅  |
| UD-Q4_K_XL | q4_0  | q4_0  | 3155   | 60     | ~20 GiB ✅  |
| UD-Q5_K_XL | f16   | f16   | 3035   | **55** | ~29 GiB ✅ ⭐ |
| UD-Q5_K_XL | q8_0  | q8_0  | 3072   | 55     | ~25 GiB ✅  |
| UD-Q5_K_XL | q4_0  | q8_0  | 3030   | 55     | ~23 GiB ✅  |
| UD-Q5_K_XL | q4_0  | q4_0  | 3044   | 55     | ~22 GiB ✅  |

KV type is noise on this model — all tg within ±2 t/s. Dense 27B is entirely weight-bound at 55–62 t/s.
Use f16 KV always (trivially fits, simplest config).

**Sweet spots:**
- `UD-Q4_K_XL + f16/f16` — 62 t/s tg, ~26 GiB, 6 GiB headroom ⭐
- `UD-Q5_K_XL + f16/f16` — 55 t/s tg, ~29 GiB, quality-first, 3 GiB headroom ⭐

**Comparison vs Qwen3.5-35B-A3B (MoE, 3B active):**
- tg: 61 vs 176 t/s (3× slower — full 27B dense vs 3B active)
- pp: 3237 vs 5939 t/s (1.8× slower)
- Trade-off: denser model → more capacity per param, slower inference

### Example Commands

**llama-cli (Q4_K_XL):**
```bash
llama-cli \
  --model /mnt/data/models/unsloth/Qwen3.5-27B-GGUF/Qwen3.5-27B-UD-Q4_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 1024 \
  --ubatch-size 1024 \
  --ctx-size 131072 \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --temp 0.7 --min-p 0.0 --top-p 0.8 --top-k 20 --presence-penalty 1.5 \
  --flash-attn 1 \
  --conversation
```

**llama-server (Q4_K_XL):**
```bash
llama-server \
  --model /mnt/data/models/unsloth/Qwen3.5-27B-GGUF/Qwen3.5-27B-UD-Q4_K_XL.gguf \
  --jinja \
  --n-gpu-layers 99 \
  --batch-size 1024 \
  --ubatch-size 1024 \
  --ctx-size 131072 \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --temp 0.7 --top-p 0.8 --top-k 20 --presence-penalty 1.5 \
  --flash-attn 1 \
  --host 0.0.0.0 --port 8080
```

### Vision Usage

> **Note:** Vision requires `llama-mtmd-cli`, NOT `llama-cli`. The standard `llama-cli` does not
> support `--mmproj` or `--image`. `llama-server` supports `--mmproj` for API-based vision.

#### llama-mtmd-cli (interactive conversation with images)
```bash
llama-mtmd-cli \
  --model /mnt/data/models/unsloth/Qwen3.5-27B-GGUF/Qwen3.5-27B-UD-Q4_K_XL.gguf \
  --mmproj /mnt/data/models/unsloth/Qwen3.5-27B-GGUF/mmproj-F16.gguf \
  --n-gpu-layers 99 \
  --ctx-size 16384 \
  --cache-type-k f16 --cache-type-v f16 \
  --flash-attn 1 \
  --temp 0.7 --top-p 0.8 --top-k 20
```
> **Note:** Do NOT use `--jinja` with `llama-mtmd-cli` — it conflicts with the vision template handling and causes a crash.
Then type your prompt. To include an image in conversation: `/path/to/image.jpg\nYour prompt here`

#### llama-server (API-based vision)
```bash
llama-server \
  --model /mnt/data/models/unsloth/Qwen3.5-27B-GGUF/Qwen3.5-27B-UD-Q4_K_XL.gguf \
  --mmproj /mnt/data/models/unsloth/Qwen3.5-27B-GGUF/mmproj-F16.gguf \
  --n-gpu-layers 99 \
  --ctx-size 16384 \
  --cache-type-k f16 --cache-type-v f16 \
  --flash-attn 1 \
  --host 0.0.0.0 --port 8080
```
