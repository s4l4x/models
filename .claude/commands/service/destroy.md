---
description: Remove a llama-server systemd service. Usage: /service:destroy [--name NAME]
---

Stop, disable, and delete a llama-server systemd user service.

## Step 1: Parse arguments

Extract `--name` from `$ARGUMENTS` (default: `llama-server`).

## Step 2: Show what will be removed

```bash
systemctl --user cat <name>.service 2>&1
```

If the unit file does not exist, tell the user there is nothing to destroy and stop.

Display the ExecStart line so the user can confirm this is the right service.

Show current status:
```bash
systemctl --user is-active <name>.service
```

## Step 3: Confirm with user

Ask: `Destroy service <name>? This will stop, disable, and delete the unit file. (y/N):`

If the user declines, stop.

## Step 4: Stop and disable

```bash
systemctl --user stop <name>.service 2>/dev/null
systemctl --user disable <name>.service 2>/dev/null
```

## Step 5: Delete the unit file

Delete `~/.config/systemd/user/<name>.service` using bash `rm`.

## Step 6: Reload and confirm

```bash
systemctl --user daemon-reload
```

Verify the service is gone:
```bash
systemctl --user status <name>.service 2>&1
```

Show VRAM:
```bash
nvidia-smi --query-gpu=memory.used,memory.free --format=csv,noheader
```

Print: `Service <name> destroyed.`
