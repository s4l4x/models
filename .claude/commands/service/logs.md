---
description: Show logs for a llama-server systemd service. Usage: /service:logs [--name NAME] [--lines N]
---

Show recent journal output for a llama-server systemd user service.

## Step 1: Parse arguments

Extract from `$ARGUMENTS`:
- `--name` (default: `llama-server`)
- `--lines` or `-n` (default: `50`)

## Step 2: Check unit file exists

```bash
systemctl --user cat <name>.service 2>&1
```

If the unit file does not exist, tell the user and stop.

## Step 3: Show logs

```bash
journalctl --user -u <name>.service -n <lines> --no-pager
```

If no output is returned, the service may never have been started — mention this.
