---
description: Download a GGUF model from HuggingFace. Usage: /download <hf-url> [pattern]
---

Download GGUF model files from HuggingFace and record the model in MODELS.md.

Arguments: $ARGUMENTS

## Steps

### 1. Parse arguments
- First arg: HuggingFace URL (e.g. https://huggingface.co/unsloth/Qwen3.5-35B-A3B-GGUF)
  Extract repo_id from the URL path (e.g. unsloth/Qwen3.5-35B-A3B-GGUF)
- Second arg (optional): file pattern (e.g. *UD-Q4_K_XL*). Default: ask the user which quant(s) to download.
- Local dir: /mnt/data/models/<org>/<repo-name>/ (derived from repo_id)

### 2. Check for existing files

Check if `/mnt/data/models/<org>/<repo-name>/` already has `.gguf` files matching the pattern.

**If local matches exist**, run a dry-run to compare against remote:
```bash
hf download <repo_id> \
  --include "<pattern>" \
  --local-dir /mnt/data/models/<org>/<repo-name>/ \
  --dry-run
```
- Files showing `-` in the "Bytes to download" column are already up-to-date (etag matches remote)
- If **all** matched files show `-`: tell the user "Already up to date — nothing to download" and stop (or offer to skip ahead to metadata update)
- If **some** files need updating: show which ones changed and ask the user to confirm before proceeding

**If no local matches exist**, proceed directly to step 3.

### 3. Download
```bash
hf download <repo_id> \
  --include "<pattern>" \
  --local-dir /mnt/data/models/<org>/<repo-name>/
```

**If the model supports vision** (i.e. has mmproj files in the repo):
- Also download the mmproj file: prefer `mmproj-F16.gguf` if available, otherwise `mmproj-F32.gguf`
- Add it to the quant inventory with status `✅ keep (required for vision)`
- Add a Vision Usage section to MODELS.md (see Gemma-3-27B-IT or Qwen3.5-35B-A3B sections as templates)

### 4. Fetch model info

**Use the `hf` CLI** (NOT python) for all metadata:
```bash
hf models info <repo_id>
```
This returns JSON with `created_at`, `last_modified`, `likes`, `tags`, `gguf` (architecture, context_length), and `siblings` (file list). Parse with `jq` as needed.

**Use WebFetch** on the HuggingFace model page and the base model page to get:
- Model summary (1-2 sentences)
- Features: tool calling, vision/multimodal, thinking/reasoning mode, context length
- Architecture: param count, MoE or dense, layers, KV heads, head dim if listed
- Recommended sampling presets (thinking vs non-thinking, coding vs general)
- Special flags (e.g. `--chat-template-kwargs` to toggle thinking mode)
- Vision mmproj filename/format (F16 vs F32)
- Any model-specific quirks or warnings

**Also check the creator's official docs** if available (e.g. `https://unsloth.ai/docs/models/<model-name>`). Look for:
- Model-specific quirks or warnings
- Recommended sampling parameters
- Special flags or chat template requirements

**Do NOT use python** for fetching metadata — the `hf` CLI and WebFetch cover everything needed.

### 5. Update MODELS.md and README.md

**5a. Add a new section to /mnt/data/models/MODELS.md** using this template:

```markdown
---

## <Model Name>

**URL:** <hf-url>
**Released:** <YYYY-MM-DD> | **Updated:** <YYYY-MM-DD> | **Likes:** <N>
**Local:** /mnt/data/models/<org>/<repo-name>/

**Summary:** <1-2 sentence summary>

**Features:**
- ✅/❌ Tool calling
- ✅/❌ Vision/multimodal
- ✅/❌ Thinking/reasoning mode
- ✅ Context: <length>

**Architecture:** <params> / <active params> | <MoE/dense> | <layers>L / <kv-heads>KVH / <head-dim>dim (confirmed/estimated)

### Quant Inventory

| file | quant | size | status |
|------|-------|------|--------|
| <filename> | <quant> | <size> | ✅ keep |

### Example Commands

**llama-cli:**
<filled in with best-guess sweet spot flags based on model size and architecture>

**llama-server:**
<same but with --host 0.0.0.0 --port 8080>
```

**5b. Append a row to the Models table in /mnt/data/models/README.md:**

```markdown
| [<Model Name>](MODELS.md#<anchor>) | <ctx> | ✅/❌ | ✅/❌ | ✅/❌ | <size> |
```

- `<anchor>` = heading slug (lowercase, spaces→hyphens, punctuation stripped)
- Columns: Model (linked), Ctx, Tool, Vision, Think, Size

### 6. Offer to bench
Ask the user: "Download complete. Run bench sweep now? (context depth + KV cache type)"
If yes, run the standard full bench from the /bench command and append results to the model's section in MODELS.md.
