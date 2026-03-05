---
description: Stop a llama-server systemd service. Usage: /service:stop [--name NAME]
---

Stop a running llama-server systemd user service.

## Step 1: Parse arguments

Extract `--name` from `$ARGUMENTS` (default: `llama-server`).

## Step 2: Check current state

```bash
systemctl --user is-active <name>.service
```

If already inactive, tell the user and stop — nothing to do.

## Step 3: Stop the service

```bash
systemctl --user stop <name>.service
```

## Step 4: Confirm and show results

Verify stopped:
```bash
systemctl --user is-active <name>.service
```

Show freed VRAM:
```bash
nvidia-smi --query-gpu=memory.used,memory.free --format=csv,noheader
```

Print: `Service <name> stopped.`
