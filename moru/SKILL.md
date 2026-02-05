---
name: moru
description: "Use this skill when the user wants to run code in an isolated cloud environment, execute untrusted code safely, build AI agents that write and run code, create reproducible development environments, or persist data across coding sessions. Triggers include: mentions of 'sandbox', 'isolated environment', 'run code safely', 'Moru', 'Firecracker', 'cloud VM', 'code execution', or requests to execute arbitrary code, install packages safely, or create custom runtime environments."
---

# Moru Cloud Sandboxes

Run code in isolated Firecracker microVMs with dedicated CPU, memory, and filesystem.

## Quick Reference

| Task | Approach |
|------|----------|
| Run a command | `sbx.commands.run("python script.py")` |
| Read/write files | `sbx.files.read()` / `sbx.files.write()` |
| Persist data | Attach a Volume to `/workspace` |
| Custom environment | Build a Template with your packages |
| Long-running server | Use `background=True`, get URL with `get_host(port)` |

## Workflow Decision

```
What are you trying to do?
│
├─ Run code once → Quick Execution (below)
├─ Keep sandbox alive → Set timeout, use keep-alive pattern
├─ Save work between sessions → Use Volumes
├─ Pre-install packages → Build a Template
└─ Debug issues → Check logs, see Troubleshooting
```

---

## Quick Execution

The most common pattern - create sandbox, run code, cleanup:

```python
from moru import Sandbox

with Sandbox.create() as sbx:
    # Write code
    sbx.files.write("/app/script.py", """
import requests
r = requests.get('https://httpbin.org/ip')
print(r.json())
""")

    # Run it
    result = sbx.commands.run("python3 /app/script.py")
    print(result.stdout)

    if result.exit_code != 0:
        print(f"Error: {result.stderr}")
# Sandbox automatically killed
```

**JavaScript equivalent:**
```typescript
import Sandbox from '@moru-ai/core'

const sbx = await Sandbox.create()
try {
  await sbx.files.write("/app/script.py", `print("Hello")`)
  const result = await sbx.commands.run("python3 /app/script.py")
  console.log(result.stdout)
} finally {
  await sbx.kill()
}
```

---

## Running Commands

### Basic execution
```python
result = sbx.commands.run("echo 'Hello'")
print(result.stdout)     # "Hello\n"
print(result.exit_code)  # 0
```

### With environment variables
```python
result = sbx.commands.run(
    "python3 -c 'import os; print(os.environ[\"API_KEY\"])'",
    envs={"API_KEY": "secret123"}
)
```

### As root (for apt-get, etc.)
```python
sbx.commands.run("apt-get update && apt-get install -y curl", user="root")
```

### Streaming output in real-time
```python
result = sbx.commands.run(
    "for i in 1 2 3; do echo $i; sleep 1; done",
    on_stdout=lambda data: print(data, end=""),
    timeout=30
)
```

### Background process (servers)
```python
# Start server
handle = sbx.commands.run("python3 -m http.server 8080", background=True)

# Get public URL
url = sbx.get_host(8080)
print(f"Server at: {url}")

# Later, stop it
handle.kill()
```

---

## Working with Files

### Write then read
```python
sbx.files.write("/app/data.json", '{"key": "value"}')
content = sbx.files.read("/app/data.json")
```

### Check existence before reading
```python
if sbx.files.exists("/app/config.json"):
    config = sbx.files.read("/app/config.json")
```

### List directory contents
```python
for entry in sbx.files.list("/app"):
    print(f"{entry.type}: {entry.name} ({entry.size} bytes)")
```

### Binary files
```python
# Write binary
with open("local.png", "rb") as f:
    sbx.files.write("/app/image.png", f.read())

# Read binary
data = sbx.files.read("/app/output.zip", format="bytes")
```

---

## Persisting Data with Volumes

Sandbox filesystems are **ephemeral** - killed sandbox = lost data. Use Volumes for persistence:

```python
from moru import Sandbox, Volume

# Create volume (idempotent - same name returns same volume)
vol = Volume.create(name="my-workspace")

# Attach to sandbox
sbx = Sandbox.create(
    volume_id=vol.volume_id,
    volume_mount_path="/workspace"  # Must be /workspace, /data, /mnt, or /volumes
)

# Anything in /workspace persists
sbx.commands.run("git clone https://github.com/user/repo /workspace")
sbx.kill()

# Later - data still there
sbx2 = Sandbox.create(volume_id=vol.volume_id, volume_mount_path="/workspace")
sbx2.commands.run("ls /workspace")  # Shows the cloned repo
```

### Access volume files without a sandbox
```python
vol = Volume.get("my-workspace")
files = vol.list_files("/")
content = vol.download("/README.md")
vol.upload("/config.json", b'{"key": "value"}')
```

---

## Custom Environments with Templates

Pre-install packages so sandboxes start instantly:

```python
from moru import Template, Sandbox
from moru.template import wait_for_port

# Define template
template = (
    Template()
    .from_python_image("3.11")
    .apt_install(["curl", "git"])
    .pip_install(["flask", "requests", "pandas"])
    .set_start_cmd("python app.py", wait_for_port(5000))
)

# Build once (takes a few minutes)
Template.build(template, alias="my-flask-app")

# Use instantly
sbx = Sandbox.create("my-flask-app")
```

### From Dockerfile
```python
template = Template().from_dockerfile("./Dockerfile")
Template.build(template, alias="my-app")
```

**CLI alternative:**
```bash
moru template create my-app --dockerfile ./Dockerfile
```

---

## Timeout Management

Default: **5 minutes**. Maximum: **1 hour** (hobby) / **24 hours** (pro).

```python
# Set at creation
sbx = Sandbox.create(timeout=3600)  # 1 hour in seconds

# Extend later
sbx.set_timeout(1800)  # Reset to 30 minutes from now
```

### Keep-alive pattern for long tasks
```python
import threading, time

def keep_alive():
    while True:
        try:
            sbx.set_timeout(600)  # Extend by 10 min
            time.sleep(300)       # Every 5 min
        except:
            break

threading.Thread(target=keep_alive, daemon=True).start()
```

---

## Common Pitfalls

### NEVER forget cleanup
```python
# ❌ WRONG - sandbox keeps running, wastes resources
sbx = Sandbox.create()
sbx.commands.run("echo hello")
# forgot to kill!

# ✅ CORRECT - use context manager
with Sandbox.create() as sbx:
    sbx.commands.run("echo hello")
# auto-killed
```

### NEVER assume packages are installed
```python
# ❌ WRONG - pandas not in base template
result = sbx.commands.run("python3 -c 'import pandas'")  # ImportError

# ✅ CORRECT - install first or use custom template
sbx.commands.run("pip install pandas", timeout=120)
result = sbx.commands.run("python3 -c 'import pandas; print(pandas.__version__)'")
```

### NEVER write outside volume mount for persistence
```python
# ❌ WRONG - /home/user is NOT persisted
sbx.files.write("/home/user/important.txt", "data")  # Lost on kill!

# ✅ CORRECT - write to volume mount path
sbx.files.write("/workspace/important.txt", "data")  # Persisted
```

### Handle command failures
```python
result = sbx.commands.run("python3 script.py")
if result.exit_code != 0:
    print(f"Failed: {result.stderr}")
    # Handle error
```

---

## Troubleshooting

### "Sandbox not found"
Sandbox was killed or timed out. Create a new one or extend timeout.

### "Command timed out"
Default is 60 seconds. Increase: `sbx.commands.run(cmd, timeout=300)`

### "Permission denied"
Use `user="root"` for system operations: `sbx.commands.run("apt-get install ...", user="root")`

### "Not enough space"
Clean up: `sbx.commands.run("rm -rf /tmp/* ~/.cache/*")`

### Debug with logs
```bash
moru sandbox logs sbx_abc123 --follow
```

---

## CLI Quick Reference

```bash
# Install
curl -fsSL https://moru.io/cli/install.sh | bash

# Login
moru auth login

# Run command in sandbox
moru sandbox run base echo 'hello'

# Create interactive sandbox
moru sandbox create python

# List/kill sandboxes
moru sandbox list
moru sandbox kill sbx_abc123

# Build template
moru template create my-app --dockerfile ./Dockerfile
```

---

## Next Steps

- For complete API reference, see [references/api.md](references/api.md)
- For more examples, see [references/examples.md](references/examples.md)
