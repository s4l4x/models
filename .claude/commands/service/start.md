---
description: Start a llama-server systemd service. Usage: /service:start [--name NAME]
---

Start an existing llama-server systemd user service and verify it comes up healthy.

## Step 1: Parse arguments

Extract `--name` from `$ARGUMENTS` (default: `llama-server`).

## Step 2: Check unit file exists

```bash
systemctl --user cat <name>.service 2>&1
```

If the unit file does not exist, tell the user to run `/service:create` first and stop.

Extract the `--port` value from the ExecStart line to know where to health-check.

## Step 3: Check current state

```bash
systemctl --user is-active <name>.service
```

If already active, tell the user it's already running, show status, and stop.

## Step 4: Start the service

```bash
systemctl --user start <name>.service
```

## Step 5: Poll for readiness

Poll the health endpoint (up to 60 s, every 2 s):
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
Print: `API ready at http://127.0.0.1:<port>/v1`

**On timeout (no UP after 60 s):**
```bash
journalctl --user -u <name>.service -n 50 --no-pager
```
Explain the failure and suggest fixes based on the journal output.
