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

### 2. Download
```bash
/mnt/data/models/.venv/bin/hf download <repo_id> \
  --include "<pattern>" \
  --local-dir /mnt/data/models/<org>/<repo-name>/
```

### 3. Fetch model info
Get the release date:
```bash
hf models info <repo_id> | jq -r '"Released: \(.created_at | split("T")[0])\nUpdated:  \(.last_modified | split("T")[0])\nLikes:    \(.likes)"'
```

Fetch the HuggingFace URL to get:
- Model summary (1-2 sentences)
- Features: tool calling, vision/multimodal, thinking/reasoning mode, context length
- Architecture: param count, MoE or dense, layers, KV heads, head dim if listed

**Also check the creator's official docs** if available (e.g. `https://unsloth.ai/docs/models/<model-name>`). Look for:
- Recommended sampling presets (thinking vs non-thinking, coding vs general)
- Special flags (e.g. `--chat-template-kwargs` to toggle thinking mode)
- Required special tokens or system prompts
- Vision mmproj filename/format (F16 vs F32)
- Any model-specific quirks or warnings

### 4. Update MODELS.md and README.md

**4a. Add a new section to /mnt/data/models/MODELS.md** using this template:

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

**4b. Append a row to the Models table in /mnt/data/models/README.md:**

```markdown
| [<Model Name>](MODELS.md#<anchor>) | <ctx> | ✅/❌ | ✅/❌ | ✅/❌ | <size> |
```

- `<anchor>` = heading slug (lowercase, spaces→hyphens, punctuation stripped)
- Columns: Model (linked), Ctx, Tool, Vision, Think, Size

### 5. Offer to bench
Ask the user: "Download complete. Run bench sweep now? (context depth + KV cache type)"
If yes, run the standard full bench from the /bench command and append results to the model's section in MODELS.md.
