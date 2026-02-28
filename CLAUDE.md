# Models Workspace

## Environment
Machine-specific config (GPU, paths, disk) is in `CONFIG.md` (gitignored).
Run `/configure` to generate it, or copy `CONFIG.md.example` and edit manually.
Model database: MODELS.md

## Model Organization
/mnt/data/models/<org>/<repo-name>/<file>.gguf — mirrors HuggingFace org structure

## Slash Commands
- /configure — detect GPU, disk, binary paths; write CONFIG.md
- /build     — clone/build llama.cpp with CUDA flags, install binaries
- /download  — download a model by HF URL, write info to MODELS.md, optionally bench
- /bench     — benchmark GGUF models with llama-bench, write results to MODELS.md
- /monitor   — snapshot VRAM, detect running llama process, suggest flag adjustments

## Building llama.cpp

```bash
sudo apt install -y build-essential cmake git curl libcurl4-openssl-dev pkg-config

cd ~
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp

cmake -B build \
    -DBUILD_SHARED_LIBS=OFF \
    -DGGML_CUDA=ON \
    -DGGML_CUDA_FA_ALL_QUANTS=ON \
    -DCMAKE_CUDA_ARCHITECTURES=120 \
    -DCMAKE_BUILD_TYPE=Release

# cmake resolves 120 → 120a (native Blackwell kernels) automatically
cmake --build build --config Release -j$(nproc) \
    --target llama-cli llama-server llama-gguf-split llama-bench llama-mtmd-cli

install -m755 build/bin/llama-* ~/.local/bin/
```

**`-DGGML_CUDA_FA_ALL_QUANTS=ON` is critical** — without it, KV cache quantization silently
falls back to f16 regardless of what you pass to `-ctk`/`-ctv`.

**`llama-mtmd-cli`** is the binary for vision/multimodal inference. `llama-cli` does NOT support
`--mmproj` / `--image`. Must rebuild to add it.

## Vision Inference
- Requires `llama-mtmd-cli` (not `llama-cli`)
- Gemma-3-27b: use `--mmproj mmproj-F32.gguf`
- Recommended sampling for Gemma: `--temp 1.0 --repeat-penalty 1.0 --min-p 0.01 --top-k 64 --top-p 0.95`
- See MODELS.md Gemma-3-27B section for full commands

## llama-bench Notes
- `--ctx-size` is NOT valid — use `--n-prompt` to simulate context depth
- Use short flags: `-ctk`/`-ctv` (not `--cache-type-k`/`--cache-type-v`) and `-fa 1` (not `--flash-attn`)
- `-ctk`/`-ctv` comma lists create ALL cross-combinations — run separate invocations for matched/asymmetric pairs
- Cannot run two bench instances simultaneously (GPU conflict) — chain with `&&`
- Unsloth UD models show as "Q8_0" in output — normal, reflects attn/embed layer blend
- `q4_0 K + q8_0 V` is the best asymmetric combo (V cache less sensitive; faster AND better quality)

## VRAM Quick Reference (32GB)

| kv type | KV @131k (Qwen MoE 4KVH) | KV @131k (Gemma 16KVH hybrid) |
|---------|--------------------------|-------------------------------|
| f16     | ~8–16 GiB                | ~10 GiB (hybrid SW cuts this) |
| q8_0    | ~4–8 GiB                 | ~5 GiB                        |
| q4_0    | ~2–4 GiB                 | ~2.6 GiB                      |

Gemma-3-27b uses hybrid sliding window (52 local layers capped at 1024 tokens, ~10 global layers
at full ctx) — KV cache is much cheaper than naive calculation suggests.
Confirmed architectures: Qwen3-Coder-30B = 48L/4KVH, Qwen3-4B = 36L/8KVH, Gemma-3-27B = 62L/16KVH(hybrid).
