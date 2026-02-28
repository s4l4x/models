# LLM Inference Workspace

Personal GGUF model workspace on RTX 5090 — download, bench, and serve local LLMs.

## Hardware

- **GPU:** RTX 5090, 32 GB VRAM
- **Storage:** 3.2 TB — models at `/mnt/data/models/`
- **Inference:** llama.cpp (`llama-cli`, `llama-server`, `llama-mtmd-cli`, `llama-bench`)

## Models

| Model | Ctx | Tool | Vision | Think | Size |
|-------|:---:|:----:|:------:|:-----:|------|
| [Qwen3-Coder-30B-A3B-Instruct](MODELS.md#qwen3-coder-30b-a3b-instruct) | 256k | ✅ | ❌ | ❌ | 16.45 GiB |
| [Qwen3.5-35B-A3B](MODELS.md#qwen35-35b-a3b) | 262k | ✅ | ✅ | ✅ | 19.16 / 23.21 GiB |
| [Gemma-3-27B-IT](MODELS.md#gemma-3-27b-it) | 128k | ✅ | ✅ | ❌ | 16 GiB + 1.6 GiB mmproj |
| [Qwen3-4B-Instruct-2507](MODELS.md#qwen3-4b-instruct-2507) | 262k | ✅ | ❌ | ❌ | 2.4 GiB |
| [Qwen3-4B-Thinking-2507](MODELS.md#qwen3-4b-thinking-2507) | 262k | ❌ | ❌ | ✅ | 2.4 GiB |
| [gpt-oss-20b](MODELS.md#gpt-oss-20b) | 131k | ✅ | ❌ | ✅ | 11 GiB |

## Directory Layout

```
/mnt/data/models/
  <org>/<repo-name>/<file>.gguf   # mirrors HuggingFace org structure
  MODELS.md                       # model database (canonical)
  CLAUDE.md                       # agent instructions (env, build, flags)
  README.md                       # this file
```

## Slash Commands (Claude Code)

| Command | Description |
|---------|-------------|
| `/download <hf-url> [pattern]` | Download a GGUF from HuggingFace, record metadata in MODELS.md, optionally bench |
| `/bench <model.gguf> [...]` | Run llama-bench sweep (context depth + KV cache types), write results to MODELS.md |
| `/monitor` | Snapshot VRAM usage, detect running llama process, suggest flag adjustments to fit in 32 GB |

## Quick Start

```bash
# 1. Download a model
/download https://huggingface.co/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF *UD-Q4_K_XL*

# 2. Serve it (copy the command from MODELS.md → Example Commands)
llama-server \
  --model /mnt/data/models/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF/Qwen3-Coder-30B-A3B-Instruct-UD-Q4_K_XL.gguf \
  --n-gpu-layers 99 --ctx-size 131072 \
  --cache-type-k f16 --cache-type-v f16 \
  --flash-attn --host 0.0.0.0 --port 8080
```

## Environment & Setup

Python venv at `.venv/` provides `huggingface_hub` and the `hf` CLI (no pip). llama.cpp binaries
live in `~/.local/bin/`. See [CLAUDE.md](CLAUDE.md) for the full build instructions, CUDA flags,
KV cache VRAM reference, and llama-bench gotchas.
