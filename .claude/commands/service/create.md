---
description: Create a systemd user service for llama-server. Usage: /service:create [--port N] [--name NAME] [--host ADDR]
---

Create a persistent, reboot-surviving llama-server managed as a systemd user service. Walk the
user through model selection and configuration using bench data from MODELS.md, write the unit
file, enable and start the service, then log the launch to HISTORY.md.

## Step 1: Parse arguments

Extract `--port` (default: 8080), `--name` (default: `llama-server`), and `--host` (default: `127.0.0.1`) from `$ARGUMENTS`. Use `0.0.0.0` when the server needs to be reachable from Docker containers or other machines on the network.

## Step 2: Read MODELS.md and build model inventory

Read `/mnt/data/models/MODELS.md`. For each `## <Model Name>` section, extract:
- **Local path** from the `**Local:**` line
- **Quant inventory table** — rows marked `✅ keep` only; separate mmproj files from model files
- **Features** (tool-use, vision, thinking, context length)
- **KV bench table** (ctk/ctv/pp/tg/total-VRAM columns)
- **Sweet spot annotation** (⭐ rows and/or "Sweet spot:" text)
- **Recommended sampling flags**
- **Special flags** (e.g., gpt-oss-20b has no `--jinja`; Gemma uses specific sampling; batch/ubatch sizes from example commands)
- **mmproj file path** if a mmproj row exists in the quant inventory with `✅ keep`

For each model, verify that the listed .gguf files actually exist on disk using `ls`:

```bash
ls <local_path>/*.gguf 2>/dev/null
```

Only include models where at least one non-mmproj .gguf is physically present.

For each present .gguf (excluding mmproj files), create one menu entry carrying all the
extracted metadata for that model+quant combination.

## Step 3: Present model menu

Print a numbered list. One line per available model+quant. Format:

```
Available models:
  1.  Qwen3-Coder-30B     UD-Q4_K_XL   16 GiB   tool-use | 256k ctx
  2.  Qwen3.5-35B         UD-Q4_K_XL   19 GiB   tool-use | vision 👁 | thinking | 262k ctx  ⭐
  3.  Qwen3.5-35B         UD-Q5_K_XL   23 GiB   tool-use | vision 👁 | thinking | 262k ctx
  4.  Gemma-3-27B         UD-Q4_K_XL   16 GiB   vision 👁 | tool-use | 128k ctx
  5.  Qwen3-4B-Instruct   UD-Q4_K_XL    2.4 GiB  tool-use | 262k ctx  (fast)
  6.  Qwen3-4B-Thinking   UD-Q4_K_XL    2.4 GiB  thinking | 262k ctx  (fast)
  7.  gpt-oss-20b         Q4_K_S       11 GiB   tool-use | reasoning | 131k ctx
```

Markers:
- 👁 if the model has a mmproj file on disk
- ⭐ if MODELS.md designates this quant as the overall sweet spot or recommended choice

Ask: "Select a model (1–N):"

## Step 4: Vision prompt (if applicable)

If the selected entry has a mmproj file on disk, ask:

```
Vision support is available (mmproj file found, adds ~0.9–1.6 GiB VRAM).
Include --mmproj for API-based vision inference? (Y/n):
```

Record whether to include mmproj and its full path.

## Step 5: Read CONFIG.md

```bash
cat /mnt/data/models/CONFIG.md
```

Extract `bin_dir` (llama binaries path) and `models_root`. If CONFIG.md is missing or lacks
these values, warn the user to run `/configure` first and stop.

## Step 6: Derive recommended config from MODELS.md bench data

Read the KV bench table for the selected model+quant from MODELS.md. Find the ⭐ row(s).
Use those as the recommended values. If no explicit ⭐ row, use the "Sweet spot:" note.

Show the recommended configuration derived from actual bench data:

```
Recommended config for <Model> <Quant>:
  --ctx-size        <ctx>
  --cache-type-k    <ctk>   ← <bench note, e.g. "f16/f16: 287 t/s, 28.5 GiB ⭐">
  --cache-type-v    <ctv>
  --batch-size      <bs>    (from example commands in MODELS.md)
  --ubatch-size     <ubs>
  <sampling flags>          (from MODELS.md recommended sampling)
  --jinja           (omit for gpt-oss-20b and Gemma)
  --flash-attn 1
  --threads         8
  --host            <host>    (127.0.0.1 = local only; 0.0.0.0 = network/Docker)
  --port            <port>

Estimated VRAM: ~<X> GiB / 32 GiB (<Y> GiB headroom)

[1] Accept recommended   [2] Customize settings
```

Important model-specific rules derived from MODELS.md:
- **gpt-oss-20b**: do NOT include `--jinja` (uses embedded chat template)
- **Gemma-3-27B**: do NOT include `--jinja`; use Gemma-specific sampling (`--temp 1.0 --repeat-penalty 1.0 --min-p 0.01 --top-k 64 --top-p 0.95`)
- **All others**: include `--jinja`
- **--flash-attn 1**: always include (required for KV quantization to take effect)
- **Qwen3.5-35B**: recommended anti-repetition is `--presence-penalty` (not `--repeat-penalty`)

## Step 7a: Accept → proceed to Step 8

## Step 7b: Customize → step through each setting interactively

Walk through each setting one at a time. Show the model-specific bench context, then prompt.
Press Enter to accept the suggested value.

**ctx-size:**
```
Context size [suggested: <X>]
  Model max: <native ctx> native
  Memory: KV scales linearly — halving ctx halves KV VRAM
  At <ctx>: ~<KV GiB> KV (from bench table)
  Bench: <pp scaling note from MODELS.md>
Enter value or press Enter to accept [<X>]:
```

**cache-type-k:**
```
KV cache — K type [suggested: <ctk>]
  Options: f16 | q8_0 | q4_0
  This model's bench (pp@4096 / tg@128 / total @<ctx>):
    <paste entire KV table from MODELS.md for this quant, with ✅/❌/⭐ markers>
  Note: V cache is less sensitive to quantization — q4_0 K + q8_0 V is the best
        asymmetric combo for most models (faster tg AND better quality than q8K+q4V)
Enter value (f16 / q8_0 / q4_0) or Enter to accept [<ctk>]:
```

**cache-type-v:**
```
KV cache — V type [suggested: <ctv>]
  (See table above. V is less sensitive to quantization than K.)
Enter value (f16 / q8_0 / q4_0) or Enter to accept [<ctv>]:
```

**Sampling flags** (temp, top-p, top-k, repeat-penalty or presence-penalty):
Show MODELS.md sampling table for this model if present. Prompt each param individually.
Sane ranges: temp 0–1.5, top-p 0–1, top-k 1–100.

**batch-size / ubatch-size:**
Show values from MODELS.md example commands. Default: 1024/1024 for large models, 2048/512
for small/dense models.

**threads:** Suggested 8. Sane 4–16. Note diminishing returns above 8 for inference.

**port:** Default from Step 1 args.

**Extra flags:**
```
Any additional flags? (e.g. --parallel 4  --mlock  --no-mmap)
Leave blank to skip:
```

After all settings, display the final assembled command for confirmation:
```
Final ExecStart:
  <bin_dir>/llama-server --model <path> [--mmproj <path>] [--jinja] \
    --n-gpu-layers 99 --batch-size <bs> --ubatch-size <ubs> \
    --ctx-size <ctx> --cache-type-k <ctk> --cache-type-v <ctv> \
    <sampling> --flash-attn 1 --threads <n> --host <host> --port <port> [extra]

Proceed? (Y/n):
```

## Step 8: Write the systemd unit file

Get the current username:
```bash
whoami
```

Create the directory if needed:
```bash
mkdir -p ~/.config/systemd/user/
```

Write `~/.config/systemd/user/<name>.service`. The ExecStart **must be a single unbroken line**
(systemd does not support backslash continuation).

Unit template:
```ini
[Unit]
Description=llama-server: <model basename>
After=network.target

[Service]
Type=simple
Environment="GGML_CUDA_ENABLE_UNIFIED_MEMORY=1"
Environment="PATH=/home/<username>/.local/bin:/usr/local/bin:/usr/bin:/bin"
ExecStart=<bin_dir>/llama-server --model <model_path> [--mmproj <mmproj_path>] [--jinja] --n-gpu-layers 99 --batch-size <bs> --ubatch-size <ubs> --ctx-size <ctx> --cache-type-k <ctk> --cache-type-v <ctv> <sampling flags> --flash-attn 1 --threads <threads> --host <host> --port <port> [extra flags]
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```

Use the Write tool to create the file (do NOT use bash heredoc/echo redirection).

## Step 9: Update HISTORY.md and .gitignore

**Ensure HISTORY.md is gitignored.** Check `/mnt/data/models/.gitignore`:
```bash
grep -q "^HISTORY\.md" /mnt/data/models/.gitignore 2>/dev/null || echo "missing"
```
If missing, append `HISTORY.md` to `/mnt/data/models/.gitignore`.

**Append to HISTORY.md.** If the file does not exist, create it with a header first:
```
# Service launch history — auto-generated by /service:create. Gitignored.
```

Then append (using the Edit or Write tool — NOT bash echo):
```
<YYYY-MM-DD HH:MM> | <full ExecStart command, single line>
```

Get the current timestamp:
```bash
date '+%Y-%m-%d %H:%M'
```

## Step 10: Enable, start, and verify

```bash
systemctl --user daemon-reload
systemctl --user enable <name>.service
systemctl --user start <name>.service
```

Poll for readiness (up to 60 s, every 2 s):
```bash
for i in $(seq 1 30); do
  curl -sf http://127.0.0.1:<port>/health && echo "UP" && break
  sleep 2
done
```

**On success:** Show:
```bash
systemctl --user status <name>.service --no-pager
nvidia-smi --query-gpu=memory.used,memory.free --format=csv,noheader
```
Then print: `API ready at http://127.0.0.1:<port>/v1`

**On timeout (no UP after 60 s):**
```bash
journalctl --user -u <name>.service -n 50 --no-pager
```
Explain the failure and suggest fixes based on the journal output.

**Offer boot persistence:**
```
To auto-start at boot (survives reboots without login), run:
  loginctl enable-linger <username>
Run this now? (y/N):
```
If yes: `loginctl enable-linger <username>`
