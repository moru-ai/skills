---
name: moru
description: Complete guide for Moru cloud sandboxes - isolated compute environments for AI agents. Use when creating/managing sandboxes, running commands, handling files, building templates, using volumes, debugging issues, or integrating Moru SDK (Python/JavaScript). Triggers on "moru", "sandbox", "moru sandbox", "moru template", "moru volume", "moru cli", "cloud sandbox", "isolated environment", "firecracker", "code execution".
---

# Moru Sandbox Guide

Moru provides isolated cloud sandboxes (Firecracker microVMs) for secure code execution. Each sandbox has dedicated CPU, memory, and disk with full Linux environment.

## Quick Start

```bash
# Install CLI
curl -fsSL https://moru.io/cli/install.sh | bash
# or: npm install -g @moru-ai/cli
# or: brew tap moru-ai/moru && brew install moru

# Login and run
moru auth login
moru sandbox run base echo 'hello world!'
```

```python
# Python SDK
from moru import Sandbox

with Sandbox.create() as sbx:
    result = sbx.commands.run("echo 'Hello!'")
    print(result.stdout)
```

```typescript
// JavaScript SDK
import Sandbox from '@moru-ai/core'

const sbx = await Sandbox.create()
const result = await sbx.commands.run("echo 'Hello!'")
console.log(result.stdout)
await sbx.kill()
```

## Default Resources

| Resource | Default | Maximum |
|----------|---------|---------|
| vCPUs | 2 | Plan-dependent |
| Memory | 512 MB | 4 GB |
| Disk | 10 GB | 10 GB |
| Timeout | 5 min | 1 hour (hobby) / 24 hours (pro) |

## Reference Documentation

Detailed guides organized by topic:

| Topic | File | Use When |
|-------|------|----------|
| **Getting Started** | [references/getting-started.md](references/getting-started.md) | Installing CLI, getting API key, first sandbox |
| **Sandbox Operations** | [references/sandbox.md](references/sandbox.md) | Create, connect, kill, timeout, lifecycle |
| **CLI Commands** | [references/cli.md](references/cli.md) | Using moru CLI for sandbox/template/volume management |
| **Templates** | [references/templates.md](references/templates.md) | Building custom templates, Dockerfile support |
| **Files & Volumes** | [references/files-volumes.md](references/files-volumes.md) | File I/O, persistent volumes, storage |
| **Command Execution** | [references/commands.md](references/commands.md) | Running commands, streaming, background processes |
| **Python SDK** | [references/sdk-python.md](references/sdk-python.md) | Python sync/async API reference |
| **JavaScript SDK** | [references/sdk-javascript.md](references/sdk-javascript.md) | TypeScript/JavaScript API reference |
| **Troubleshooting** | [references/troubleshooting.md](references/troubleshooting.md) | Common errors, debugging, limits |

## Common Patterns

### Run Command and Get Output
```python
result = sbx.commands.run("python3 script.py")
print(result.stdout)  # stdout
print(result.stderr)  # stderr
print(result.exit_code)  # 0 = success
```

### File Operations
```python
sbx.files.write("/app/data.json", '{"key": "value"}')
content = sbx.files.read("/app/data.json")
entries = sbx.files.list("/app")
```

### Background Process
```python
handle = sbx.commands.run("python3 server.py", background=True)
# ... do other work ...
result = handle.wait()
```

### Persistent Storage with Volumes
```python
from moru import Sandbox, Volume

vol = Volume.create(name="workspace")
sbx = Sandbox.create(
    volume_id=vol.volume_id,
    volume_mount_path="/workspace"
)
# Data in /workspace persists after sandbox kill
```

### Custom Template
```python
from moru import Template

template = (
    Template()
    .from_python_image("3.11")
    .pip_install(["requests", "flask"])
    .set_start_cmd("python app.py")
)
Template.build(template, alias="my-app")
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `MORU_API_KEY` | API key for SDK/CLI authentication |
| `MORU_ACCESS_TOKEN` | Access token from `moru auth login` |
| `MORU_DOMAIN` | API domain (default: moru.io) |

## Tips

- Use context managers (`with Sandbox.create() as sbx:`) for automatic cleanup
- Set appropriate timeouts for long-running tasks
- Use templates to pre-install dependencies for faster startup
- Use volumes for data that needs to persist across sandbox restarts
- Check `sbx.is_running()` before operations on potentially terminated sandboxes
