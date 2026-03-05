---
description: Benchmark one or more GGUF models with llama-bench. Usage: /bench <model.gguf> [model2.gguf ...]
---

Run llama-bench on one or more GGUF models and produce a consolidated results table.

Arguments (model paths): $ARGUMENTS

## Rules
- Use SHORT flags only: `-p` (not `--n-prompt`), `-n` (not `--n-gen`), `-ctk`/`-ctv`, `-fa 1`, `-ngl`, `-t`
- `-ctk`/`-ctv` comma lists create ALL cross-combinations — run separate invocations for matched/asymmetric pairs
- Cannot run two llama-bench instances simultaneously (GPU conflict) — chain with `&&`
- Unsloth UD models show as "Q8_0" in output — normal, reflects attn/embed layer quant blend
- **CRITICAL:** Run pp and tg as SEPARATE invocations. Using `-p 512,4096,16384,65536 -n 128` in one command creates cross-products (tg at every context size), wasting hours on useless tg65536 runs.

## Standard Full Bench (run sequentially, chain with &&)

### 1. Context depth sweep (q8_0 KV baseline)

**Prompt processing** (pp-only, `-n 0`):
```bash
llama-bench \
  -m <model1> [-m <model2> ...] \
  -p 512,4096,16384,65536 -n 0 \
  -ngl 99 -fa 1 -ctk q8_0 -ctv q8_0 \
  -t 8 -r 3
```

**Token generation** (tg-only, `-p 0`):
```bash
llama-bench \
  -m <model1> [-m <model2> ...] \
  -p 0 -n 128 \
  -ngl 99 -fa 1 -ctk q8_0 -ctv q8_0 \
  -t 8 -r 3
```

### 2. KV cache type comparison (pp@4096 + tg@128)

Run each KV combo as TWO separate invocations (pp then tg). Already have q8_0/q8_0 from step 1.

**For each of: f16/f16, q4_0/q8_0, q4_0/q4_0:**
```bash
llama-bench -m <model> -ngl 99 -fa 1 -t 8 -r 3 \
  -ctk <K> -ctv <V> -p 4096 -n 0 && \
llama-bench -m <model> -ngl 99 -fa 1 -t 8 -r 3 \
  -ctk <K> -ctv <V> -p 0 -n 128
```

Note: q4_0 K + q8_0 V is the best asymmetric combo — faster tg AND better quality than q8K+q4V.

## Output
After all runs complete, consolidate into two tables and write them to the model's section in MODELS.md:

**Table 1 — Context depth** (rows = quant×ctx, cols = pp t/s, tg t/s)

**Table 2 — KV cache type** (rows = quant×ctk×ctv, cols = pp t/s, tg t/s, total @131k, fits?)

Estimate "total @131k" from model file size + KV cache calculation:
- KV bytes per token per layer = 2 × n_kv_heads × head_dim × bytes_per_element
- Multiply by n_attention_layers (NOT total layers for hybrid models) × 131072

Then:
- Call out the sweet spot: best tg that fits comfortably in 32GB at 131k context
- Update the model's Example Commands in MODELS.md to use the sweet spot KV flags
- Note any architectural findings (e.g. confirmed layer/KV-head count from VRAM observations)
