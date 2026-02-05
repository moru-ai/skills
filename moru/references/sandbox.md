# Sandbox Operations

## Table of Contents
- [Sandbox Lifecycle](#sandbox-lifecycle)
- [Create Sandbox](#create-sandbox)
- [Connect to Sandbox](#connect-to-sandbox)
- [Sandbox Information](#sandbox-information)
- [Timeout Management](#timeout-management)
- [Kill Sandbox](#kill-sandbox)
- [List Sandboxes](#list-sandboxes)
- [Network Configuration](#network-configuration)
- [Environment Variables](#environment-variables)
- [Metadata](#metadata)

---

## Sandbox Lifecycle

```
[*] → Creating → Running → Terminated
                    ↓
                  Paused (beta)
```

**States:**
- **Running**: Active, accepting operations
- **Paused** (beta): Suspended, filesystem preserved
- **Terminated**: Stopped, resources released

---

## Create Sandbox

### Python

```python
from moru import Sandbox

# Default 'base' template
sandbox = Sandbox.create()

# Specific template
sandbox = Sandbox.create("python")

# With all options
sandbox = Sandbox.create(
    template="python",
    timeout=600,              # 10 minutes (seconds)
    metadata={"project": "my-agent", "user_id": "123"},
    envs={"API_KEY": "secret", "DEBUG": "true"},
    allow_internet_access=True,
    secure=True,
    volume_id="vol_abc123",
    volume_mount_path="/workspace",
    network={
        "allow_out": ["api.openai.com"],
        "deny_out": ["internal.company.com"],
    }
)
```

### JavaScript

```typescript
import Sandbox from '@moru-ai/core'

// Default
const sandbox = await Sandbox.create()

// Specific template
const sandbox = await Sandbox.create("python")

// With options
const sandbox = await Sandbox.create("python", {
  timeoutMs: 600000,          // 10 minutes (milliseconds)
  metadata: { project: "my-agent" },
  envs: { API_KEY: "secret" },
  allowInternetAccess: true,
  secure: true,
  volumeId: "vol_abc123",
  volumeMountPath: "/workspace",
  network: {
    allowOut: ["api.openai.com"],
    denyOut: [],
  }
})
```

### Create Options Reference

| Option (Python) | Option (JS) | Type | Default | Description |
|-----------------|-------------|------|---------|-------------|
| `template` | `template` | string | `"base"` | Template ID or name |
| `timeout` | `timeoutMs` | number | 300s / 300000ms | Auto-termination timeout |
| `metadata` | `metadata` | object | `{}` | Custom key-value pairs |
| `envs` | `envs` | object | `{}` | Environment variables |
| `allow_internet_access` | `allowInternetAccess` | boolean | `true` | Enable outbound network |
| `network` | `network` | object | `null` | Fine-grained network control |
| `secure` | `secure` | boolean | `true` | Enable security features |
| `volume_id` | `volumeId` | string | `null` | Volume to attach |
| `volume_mount_path` | `volumeMountPath` | string | required with volume | Mount path |

---

## Connect to Sandbox

Reconnect to an existing running sandbox.

### Python

```python
from moru import Sandbox

# Connect by ID
sandbox = Sandbox.connect("sbx_abc123...")

# Check if still running
if sandbox.is_running():
    result = sandbox.commands.run("echo 'Reconnected!'")
```

### JavaScript

```typescript
import Sandbox from '@moru-ai/core'

const sandbox = await Sandbox.connect("sbx_abc123...")

if (await sandbox.isRunning()) {
  const result = await sandbox.commands.run("echo 'Reconnected!'")
}
```

### Session Recovery Pattern

```python
import json
from moru import Sandbox

# Save sandbox ID
sandbox = Sandbox.create()
with open("session.json", "w") as f:
    json.dump({"sandbox_id": sandbox.sandbox_id}, f)

# Later, reconnect
with open("session.json") as f:
    data = json.load(f)
sandbox = Sandbox.connect(data["sandbox_id"])
```

---

## Sandbox Information

### Python

```python
sandbox = Sandbox.create()

# Basic properties
print(sandbox.sandbox_id)      # "sbx_abc123..."

# Detailed info
info = sandbox.get_info()
print(f"Template: {info.template_id}")
print(f"Started: {info.started_at}")
print(f"Ends: {info.end_at}")
print(f"State: {info.state}")
print(f"CPUs: {info.cpu_count}")
print(f"Memory: {info.memory_mb} MB")
```

### JavaScript

```typescript
const sandbox = await Sandbox.create()

console.log(sandbox.sandboxId)

const info = await sandbox.getInfo()
console.log(`Template: ${info.templateId}`)
console.log(`State: ${info.state}`)
```

### Get Metrics

```python
metrics = sandbox.get_metrics()
for m in metrics:
    print(f"CPU: {m.cpu_used_pct:.1f}%")
    print(f"Memory: {m.mem_used / 1024 / 1024:.0f} MB / {m.mem_total / 1024 / 1024:.0f} MB")
    print(f"Disk: {m.disk_used / 1024 / 1024:.0f} MB / {m.disk_total / 1024 / 1024:.0f} MB")
```

---

## Timeout Management

### Default Timeout
- **5 minutes (300 seconds)**
- Maximum: **1 hour** (hobby) / **24 hours** (pro)

### Set Custom Timeout

```python
# At creation (seconds)
sandbox = Sandbox.create(timeout=600)  # 10 minutes

# Extend later
sandbox.set_timeout(1800)  # 30 minutes from now

# Static method
Sandbox.set_timeout("sbx_abc123...", 1800)
```

```typescript
// At creation (milliseconds)
const sandbox = await Sandbox.create({ timeoutMs: 600000 })

// Extend later
await sandbox.setTimeout(1800000)
```

### Check Remaining Time

```python
from datetime import datetime

info = sandbox.get_info()
remaining = info.end_at - datetime.now(info.end_at.tzinfo)
print(f"Expires in {remaining.total_seconds()} seconds")
```

### Keep-Alive Pattern

```python
import threading
import time

sandbox = Sandbox.create()

def keep_alive():
    while True:
        try:
            sandbox.set_timeout(600)  # Extend by 10 min
            time.sleep(300)           # Check every 5 min
        except Exception:
            break

thread = threading.Thread(target=keep_alive, daemon=True)
thread.start()
```

---

## Kill Sandbox

### Python

```python
# Instance method
sandbox.kill()

# Static method (by ID)
Sandbox.kill("sbx_abc123...")

# Context manager (auto-kill)
with Sandbox.create() as sandbox:
    result = sandbox.commands.run("echo 'Hello'")
# Automatically killed
```

### JavaScript

```typescript
await sandbox.kill()

// Or by ID
await Sandbox.kill("sbx_abc123...")
```

### Graceful Shutdown

```python
# Stop processes gracefully before killing
sandbox.commands.run("pkill -SIGTERM python3")
time.sleep(2)
sandbox.kill()
```

### What Happens on Kill

1. All running processes terminated immediately
2. Filesystem deleted (unless paused)
3. Network connections closed
4. Resources (CPU, memory, disk) released
5. Sandbox ID becomes invalid

---

## List Sandboxes

### Python

```python
# List all running sandboxes
for info in Sandbox.list():
    print(f"{info.sandbox_id}: {info.state}")

# With filters
for info in Sandbox.list(metadata={"project": "my-agent"}):
    print(info.sandbox_id)
```

### JavaScript

```typescript
const paginator = Sandbox.list()
for await (const info of paginator) {
  console.log(`${info.sandboxId}: ${info.state}`)
}
```

### Kill Multiple Sandboxes

```python
for info in Sandbox.list():
    if info.metadata.get("environment") == "development":
        Sandbox.kill(info.sandbox_id)
```

---

## Network Configuration

### Disable Internet Access

```python
sandbox = Sandbox.create(allow_internet_access=False)
```

### Allow Specific Destinations

```python
sandbox = Sandbox.create(
    network={
        "allow_out": ["api.openai.com", "8.8.8.8/32"],
        "deny_out": [],
        "allow_public_traffic": True,
    }
)
```

### Block Specific Destinations

```python
sandbox = Sandbox.create(
    network={
        "deny_out": ["internal.company.com"],
    }
)
```

### Expose Ports

```python
sandbox = Sandbox.create()
sandbox.commands.run("python3 -m http.server 8080", background=True)
host = sandbox.get_host(8080)  # https://host.moru.io
print(f"Access at: {host}")
```

---

## Environment Variables

### At Creation

```python
sandbox = Sandbox.create(
    envs={
        "OPENAI_API_KEY": "sk-...",
        "DATABASE_URL": "postgres://...",
        "NODE_ENV": "production",
    }
)

result = sandbox.commands.run("echo $OPENAI_API_KEY")
```

### Per-Command

```python
result = sandbox.commands.run(
    "echo $MY_VAR",
    envs={"MY_VAR": "Hello"}
)
```

---

## Metadata

Tag sandboxes with custom metadata for filtering and tracking.

```python
sandbox = Sandbox.create(
    metadata={
        "user_id": "user_123",
        "project": "my-agent",
        "environment": "production",
        "session_id": "sess_abc",
    }
)

# List by metadata
for info in Sandbox.list(metadata={"project": "my-agent"}):
    print(info.sandbox_id)
```

---

## Pause/Resume (Beta)

```python
sandbox = Sandbox.create()
sandbox.files.write("/data/important.txt", "Preserve this!")

# Pause - filesystem preserved
sandbox.beta_pause()

# Later, resume
sandbox = Sandbox.connect("sbx_abc123...")
sandbox.connect()  # Resume from paused

content = sandbox.files.read("/data/important.txt")  # Still there!
```

---

## Error Handling

```python
from moru import Sandbox
from moru.exceptions import (
    NotFoundException,
    TimeoutException,
    AuthenticationException,
)

try:
    sandbox = Sandbox.create("my-template")
except AuthenticationException:
    print("Invalid API key")
except NotFoundException:
    print("Template not found")
except TimeoutException:
    print("Sandbox creation timed out")
```

See [Troubleshooting](troubleshooting.md) for more error handling patterns.
