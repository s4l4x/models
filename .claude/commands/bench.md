---
description: Benchmark one or more GGUF models with llama-bench. Usage: /bench <model.gguf> [model2.gguf ...]
---

Run llama-bench on one or more GGUF models and produce a consolidated results table.

Arguments (model paths): $ARGUMENTS

## Rules
- `--ctx-size` is NOT a valid llama-bench flag — use `--n-prompt` to simulate context depth
- `-ctk`/`-ctv` comma lists create ALL cross-combinations — run separate invocations for matched/asymmetric pairs
- Cannot run two llama-bench instances simultaneously (GPU conflict) — chain with `&&`
- Unsloth UD models show as "Q8_0" in output — normal, reflects attn/embed layer quant blend

## Standard Full Bench (run sequentially, chain with &&)

### 1. Context depth sweep (q8_0 KV baseline)
```bash
llama-bench \
  -m <model1> [-m <model2> ...] \
  --n-prompt 512,4096,16384,65536 \
  --n-gen 128 \
  --batch-size 2048 \
  --ubatch-size 512 \
  --cache-type-k q8_0 --cache-type-v q8_0 \
  --n-gpu-layers 99 \
  --flash-attn 1 \
  --output md
```

### 2. KV cache type comparison (fixed pp@4096) — run 4 separate invocations:
- `--cache-type-k f16   --cache-type-v f16`
- `--cache-type-k q8_0  --cache-type-v q8_0`
- `--cache-type-k q4_0  --cache-type-v q8_0`   ← preferred asymmetric (q4K+q8V)
- `--cache-type-k q4_0  --cache-type-v q4_0`

Note: q4_0 K + q8_0 V is the best asymmetric combo — faster tg AND better quality than q8K+q4V.

## Output
After all runs complete, consolidate into two tables and write them to the model's section in MODELS.md:

**Table 1 — Context depth** (rows = model×ctx, cols = pp t/s, tg t/s)

**Table 2 — KV cache type** (rows = model×ctk×ctv, cols = pp t/s, tg t/s, total @131k, fits?)

Then:
- Call out the sweet spot: best tg that fits comfortably in 32GB at 131k context
- Update the model's Example Commands in MODELS.md to use the sweet spot KV flags
- Note any architectural findings (e.g. confirmed layer/KV-head count from VRAM observations)
