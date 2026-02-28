# Models Workspace

## Environment
- GPU: RTX 5090, 32GB VRAM
- Python venv: /mnt/data/models/.venv (huggingface_hub, hf CLI — no pip)
- System python3 also has huggingface_hub 1.5.0
- llama-bench / llama-cli / llama-server: /usr/local/bin/
- Models root: /mnt/data/models/ (3.2TB free)
- Model database: MODELS.md

## Model Organization
/mnt/data/models/<org>/<repo-name>/<file>.gguf — mirrors HuggingFace org structure

## Slash Commands
- /download — download a model by HF URL, write info to MODELS.md, optionally bench
- /bench    — benchmark GGUF models with llama-bench, write results to MODELS.md

## llama-bench Notes
- `--ctx-size` is NOT valid — use `--n-prompt` to simulate context depth
- `-ctk`/`-ctv` comma lists create ALL cross-combinations — run separate invocations for matched/asymmetric pairs
- Cannot run two bench instances simultaneously (GPU conflict) — chain with `&&`
- Unsloth UD models show as "Q8_0" in output — normal, reflects attn/embed layer blend
- `q4_0 K + q8_0 V` is the best asymmetric combo (V cache less sensitive; faster AND better quality than q8K+q4V)

## VRAM Quick Reference (32GB)

| kv type | KV @131k (est.) | safe model budget |
|---------|-----------------|-------------------|
| f16     | 12–16 GiB       | 16–20 GiB         |
| q8_0    | 6–8 GiB         | 24–26 GiB         |
| q4_0    | 3–4 GiB         | 28–29 GiB         |

KV cache size depends on model architecture (layers × KV heads × head_dim × ctx).
Confirmed: Qwen3-Coder-30B = 48L/4KVH/128dim. Qwen3.5-35B architecture unconfirmed (est. 64L/4KVH/128dim).
