---
description: Clone and build llama.cpp with CUDA/Blackwell flags, then install binaries. Usage: /build
---

Build llama.cpp from source with the correct CUDA flags for this machine and install the binaries.

## Steps

### 1. Install system dependencies

```bash
sudo apt install -y build-essential cmake git curl libcurl4-openssl-dev pkg-config
```

### 2. Clone or update llama.cpp

If `~/llama.cpp` does not exist:
```bash
git clone https://github.com/ggml-org/llama.cpp ~/llama.cpp
```

If it already exists:
```bash
git -C ~/llama.cpp pull
```

### 3. Configure

```bash
cd ~/llama.cpp

cmake -B build \
    -DBUILD_SHARED_LIBS=OFF \
    -DGGML_CUDA=ON \
    -DGGML_CUDA_FA_ALL_QUANTS=ON \
    -DCMAKE_CUDA_ARCHITECTURES=120 \
    -DCMAKE_BUILD_TYPE=Release
```

**Critical flags — do not omit:**
- `-DGGML_CUDA_FA_ALL_QUANTS=ON` — without this, KV cache quantization (`-ctk`/`-ctv`) silently falls back to f16
- `-DCMAKE_CUDA_ARCHITECTURES=120` — targets Blackwell (RTX 5090); cmake resolves 120 → 120a automatically

### 4. Build

```bash
cmake --build ~/llama.cpp/build --config Release -j$(nproc) \
    --target llama-cli llama-server llama-gguf-split llama-bench llama-mtmd-cli
```

### 5. Install

```bash
install -m755 ~/llama.cpp/build/bin/llama-* ~/.local/bin/
```

### 6. Verify

```bash
llama-bench --version
```

Print the installed binary versions and confirm they are in PATH.

### 7. Notes

- `llama-mtmd-cli` is required for vision/multimodal inference (`--mmproj`, `--image`). `llama-cli` does not support these flags.
- If the CUDA architecture differs from 120 (RTX 5090/Blackwell), update `-DCMAKE_CUDA_ARCHITECTURES` accordingly (e.g. 89 for RTX 4090, 86 for RTX 3090).
- After a rebuild, re-run `/configure` if binary paths changed.
