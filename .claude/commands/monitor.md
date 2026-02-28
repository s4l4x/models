---
description: Snapshot VRAM usage, detect running llama process, and suggest tuning flags to fit within 32GB. Usage: /monitor
---

Analyze current GPU memory pressure and recommend flag adjustments.

## Steps

### 1. Snapshot VRAM and running process

Run these two commands in parallel:

```bash
nvidia-smi --query-gpu=memory.used,memory.total,memory.free --format=csv,noheader,nounits
```

```bash
ps aux | grep -E 'llama-(cli|server|mtmd-cli)' | grep -v grep
```

### 2. Parse state

From nvidia-smi output:
- `used_mib`, `total_mib`, `free_mib`
- Pressure = used / total × 100%

From the process line, extract current flags (best-effort):
- `-m` / `--model` → model file path and name
- `--ctx-size` or `-c` → current context
- `--cache-type-k` / `-ctk` → K cache type
- `--cache-type-v` / `-ctv` → V cache type
- `--n-gpu-layers` / `-ngl` → GPU layers

### 3. Report current state

Print a status block:

```
═══════════════════════════════════════════
  VRAM Monitor — RTX 5090 (32 GB)
═══════════════════════════════════════════
  Used:    <used> MiB / <total> MiB  (<pct>%)
  Free:    <free> MiB
  Pressure: [LOW | MODERATE | HIGH | CRITICAL]

  Running: <binary> (<model short name>)
  ctx:     <ctx-size>    ctk: <K type>   ctv: <V type>
  ngl:     <ngl>
═══════════════════════════════════════════
```

Pressure thresholds:
- LOW: free > 6 GB
- MODERATE: free 3–6 GB
- HIGH: free 1–3 GB
- CRITICAL: free < 1 GB (OOM risk)

If no llama process is running, report idle VRAM and skip tuning suggestions.

### 4. Suggest tuning (if MODERATE, HIGH, or CRITICAL)

Show an ordered action table from cheapest → most impactful. Use the
architecture notes from CLAUDE.md / MEMORY.md to estimate savings.

**KV cache savings reference (Qwen3 MoE 4KVH @ full ctx):**
| Change | Est. savings |
|--------|-------------|
| ctk: f16 → q8_0 | ~4 GiB |
| ctk: q8_0 → q4_0 | ~2 GiB |
| ctv: f16 → q8_0 | ~4 GiB |
| ctv: q8_0 → q4_0 | ~2 GiB |
| ctx 131k → 65k | ~50% KV |
| ctx 65k → 32k | ~50% KV |
| ngl 99 → 80 (8 layers on CPU) | ~1–3 GiB |

**Output example:**
```
Suggested mitigations (in order of preference):

  1. Lower K cache:  add --cache-type-k q4_0   → saves ~2 GiB
  2. Lower ctx:      change --ctx-size to 65536 → saves ~3 GiB
  3. CPU offload:    change --n-gpu-layers to 80 → saves ~1-2 GiB
                     (or use --moe-expert-used-ngl N for MoE partial offload)
```

Tailor the suggestions to what's actually extracted from the process flags:
- Skip suggestions that are already applied (e.g. already on q4_0)
- Note if the model is MoE (use `--moe-expert-used-ngl` instead of blunt `-ngl` reduction)
- If CRITICAL and multiple steps needed, show cumulative savings

### 5. Reconstruct command

If suggestions exist, print the full revised launch command with the
recommended flags substituted in, ready to copy-paste. Base it on the
detected process command line.

### 6. Offer to watch

Ask: "Run live watch? (`watch -n2 nvidia-smi dmon -s mu` streams MiB/utilization)"
If yes:
```bash
watch -n2 'nvidia-smi --query-gpu=memory.used,memory.free,utilization.gpu --format=csv,noheader'
```
