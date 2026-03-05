---
description: Show status of a llama-server systemd service. Usage: /service:info [--name NAME]
---

Show the current state of a llama-server systemd user service.

## Step 1: Parse arguments

Extract `--name` from `$ARGUMENTS` (default: `llama-server`).

## Step 2: Check unit file exists

```bash
systemctl --user cat <name>.service 2>&1
```

If the unit file does not exist, tell the user and stop.

Extract the `--port` value from the ExecStart line.

## Step 3: Show service status

```bash
systemctl --user status <name>.service --no-pager
```

## Step 4: Show VRAM usage

```bash
nvidia-smi --query-gpu=memory.used,memory.free --format=csv,noheader
```

## Step 5: Health check (if active)

If the service is active, hit the health endpoint:
```bash
curl -sf http://127.0.0.1:<port>/health
```

If healthy, print: `API ready at http://127.0.0.1:<port>/v1`
If unhealthy or unreachable, note it and suggest checking logs with `/service:logs`.
