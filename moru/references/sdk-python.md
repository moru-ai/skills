# Python SDK Reference

## Table of Contents
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Sandbox Class](#sandbox-class)
- [AsyncSandbox Class](#asyncsandbox-class)
- [Files Module](#files-module)
- [Commands Module](#commands-module)
- [PTY Module](#pty-module)
- [Template Class](#template-class)
- [Volume Class](#volume-class)
- [Exceptions](#exceptions)

---

## Installation

```bash
pip install moru
```

**Requirements:** Python 3.8+

---

## Quick Start

```python
from moru import Sandbox

# Create sandbox
sandbox = Sandbox.create()

# Run command
result = sandbox.commands.run("echo 'Hello'")
print(result.stdout)

# File operations
sandbox.files.write("/tmp/test.txt", "Hello")
content = sandbox.files.read("/tmp/test.txt")

# Cleanup
sandbox.kill()
```

### Context Manager

```python
with Sandbox.create() as sandbox:
    result = sandbox.commands.run("python3 --version")
    print(result.stdout)
# Auto-cleanup
```

---

## Sandbox Class

### Creation

```python
from moru import Sandbox

# Default
sandbox = Sandbox.create()

# With template
sandbox = Sandbox.create("python")

# Full options
sandbox = Sandbox.create(
    template="python",
    timeout=600,                    # seconds
    metadata={"key": "value"},
    envs={"VAR": "value"},
    allow_internet_access=True,
    secure=True,
    volume_id="vol_abc",
    volume_mount_path="/workspace",
    network={"allow_out": ["api.example.com"]},
    api_key="moru_...",             # or use MORU_API_KEY env
)
```

### Connection

```python
# Connect to existing
sandbox = Sandbox.connect("sbx_abc123")
sandbox = Sandbox.connect("sbx_abc123", timeout=30)

# Check status
is_running = sandbox.is_running()
```

### Properties

```python
sandbox.sandbox_id          # "sbx_abc123..."
sandbox.sandbox_domain      # Domain
sandbox.traffic_access_token # Auth token
```

### Methods

```python
# Get info
info = sandbox.get_info()
# Returns: SandboxInfo with template_id, started_at, end_at, state, cpu_count, memory_mb

# Get metrics
metrics = sandbox.get_metrics()
# Returns: list of Metrics with cpu_used_pct, mem_used, mem_total, disk_used, disk_total

# Set/extend timeout
sandbox.set_timeout(1800)  # seconds

# Get host URL for port
host = sandbox.get_host(8080)  # Returns public URL

# Upload/download URLs
upload_url = sandbox.upload_url("/path/to/file")
download_url = sandbox.download_url("/path/to/file")

# Kill
sandbox.kill()

# Pause (beta)
sandbox.beta_pause()
```

### Static Methods

```python
# List sandboxes
for info in Sandbox.list():
    print(info.sandbox_id)

# With filters
for info in Sandbox.list(metadata={"project": "myapp"}):
    print(info.sandbox_id)

# Kill by ID
Sandbox.kill("sbx_abc123")

# Set timeout by ID
Sandbox.set_timeout("sbx_abc123", 1800)
```

---

## AsyncSandbox Class

Async version of Sandbox for asyncio.

```python
import asyncio
from moru import AsyncSandbox

async def main():
    sandbox = await AsyncSandbox.create()

    result = await sandbox.commands.run("echo 'Hello'")
    print(result.stdout)

    await sandbox.files.write("/tmp/test.txt", "content")
    content = await sandbox.files.read("/tmp/test.txt")

    await sandbox.kill()

asyncio.run(main())
```

### Async Context Manager

```python
async with await AsyncSandbox.create() as sandbox:
    result = await sandbox.commands.run("echo 'Hello'")
```

---

## Files Module

Access via `sandbox.files`

### Read

```python
# Text (default)
content = sandbox.files.read("/path/to/file.txt")
content = sandbox.files.read("/path/to/file.txt", format="text")

# Bytes
data = sandbox.files.read("/path/to/file.bin", format="bytes")

# Stream
stream = sandbox.files.read("/path/to/large.bin", format="stream")
for chunk in stream:
    process(chunk)

# As different user
content = sandbox.files.read("/etc/shadow", user="root")

# With timeout
content = sandbox.files.read("/file.txt", request_timeout=30)
```

### Write

```python
# Text
sandbox.files.write("/path/to/file.txt", "content")

# Bytes
sandbox.files.write("/path/to/file.bin", b"\x00\x01\x02")

# Multiple files
sandbox.files.write_files([
    {"path": "/file1.txt", "data": "content1"},
    {"path": "/file2.txt", "data": "content2"},
])

# As different user
sandbox.files.write("/etc/config", "data", user="root")
```

### Other Operations

```python
# List directory
entries = sandbox.files.list("/path/to/dir")
entries = sandbox.files.list("/path/to/dir", depth=5)  # Recursive

# Check existence
exists = sandbox.files.exists("/path/to/file")

# Get info
info = sandbox.files.get_info("/path/to/file")
# Returns: name, path, type, size, mode, permissions, owner, group, modified_time

# Delete
sandbox.files.remove("/path/to/file")

# Rename/move
sandbox.files.rename("/old/path", "/new/path")

# Create directory
sandbox.files.make_dir("/path/to/dir")

# Watch directory
handle = sandbox.files.watch_dir("/path/to/dir")
for event in handle.events():
    print(f"{event.type}: {event.name}")
handle.stop()
```

---

## Commands Module

Access via `sandbox.commands`

### Run

```python
# Basic
result = sandbox.commands.run("echo 'Hello'")
# Returns: CommandResult with stdout, stderr, exit_code

# With options
result = sandbox.commands.run(
    "python script.py",
    cwd="/app",
    user="root",
    envs={"DEBUG": "true"},
    timeout=120,
    on_stdout=lambda data: print(data, end=""),
    on_stderr=lambda data: print(data, end=""),
)

# Background
handle = sandbox.commands.run("python server.py", background=True)
handle.send_stdin("input\n")
result = handle.wait()
handle.kill()
```

### Process Management

```python
# List running processes
processes = sandbox.commands.list()
for proc in processes:
    print(f"{proc.pid}: {proc.command}")

# Kill process
sandbox.commands.kill(pid)
```

---

## PTY Module

Access via `sandbox.pty`

```python
# Create PTY
handle = sandbox.pty.create(
    cols=80,
    rows=24,
    on_data=lambda data: print(data.decode(), end="")
)
handle.send_input("ls -la\n")
handle.resize(120, 40)
handle.kill()

# Start with command
handle = sandbox.pty.start(
    "python3",
    cols=80,
    rows=24,
    on_data=print
)
```

---

## Template Class

```python
from moru import Template

# Build template
template = (
    Template()
    .from_python_image("3.11")
    .pip_install(["requests", "flask"])
    .copy("./app", "/app")
    .set_workdir("/app")
    .set_start_cmd("python app.py")
)

# Synchronous build
info = Template.build(
    template,
    alias="my-template",
    cpu_count=2,
    memory_mb=1024,
    on_build_logs=lambda entry: print(entry)
)

# Background build
info = Template.build_in_background(template, alias="my-template")
status = Template.get_build_status(info)
```

### Builder Methods

```python
# Base images
.from_python_image("3.11")
.from_node_image("lts")
.from_ubuntu_image("22.04")
.from_base_image()
.from_image("nginx:alpine")
.from_dockerfile("./Dockerfile")
.from_template("existing-template")

# Packages
.pip_install(["package1", "package2"])
.npm_install(["package1"])
.bun_install(["package1"])
.apt_install(["package1"])

# Files
.copy("./src", "/app/src")
.make_dir("/app/data")

# Configuration
.run_cmd("echo 'building'")
.set_workdir("/app")
.set_user("appuser")
.set_envs({"NODE_ENV": "production"})

# Startup
.set_start_cmd("python app.py")
.set_start_cmd("node server.js", "curl localhost:3000/health")
```

### Ready Commands

```python
from moru.template import wait_for_port, wait_for_url, wait_for_file, wait_for_process

.set_start_cmd("python app.py", wait_for_port(8080))
.set_start_cmd("npm start", wait_for_url("http://localhost:3000"))
.set_start_cmd("bash start.sh", wait_for_file("/tmp/ready"))
.set_start_cmd("service start", wait_for_process("myapp"))
```

---

## Volume Class

```python
from moru import Volume

# Create
vol = Volume.create(name="my-workspace")

# Get
vol = Volume.get("vol_abc123")
vol = Volume.get("my-workspace")

# List
for vol in Volume.list():
    print(f"{vol.name}: {vol.total_size_bytes} bytes")

# Properties
vol.volume_id
vol.name
vol.total_size_bytes
vol.total_file_count
vol.created_at
vol.updated_at

# File operations
files = vol.list_files("/")
files, next_token = vol.list_files("/", limit=100)

content = vol.download("/path/to/file")
vol.upload("/path/to/file", b"content")
vol.delete("/path/to/file")
vol.delete("/path/to/dir", recursive=True)

# Delete volume
vol.delete()
```

---

## Exceptions

```python
from moru.exceptions import (
    SandboxException,           # Base class
    TimeoutException,           # Operation timed out
    NotFoundException,          # Resource not found
    AuthenticationException,    # Invalid API key
    NotEnoughSpaceException,    # Disk full
    CommandExitException,       # Non-zero exit code
    TemplateException,          # Template error
    RateLimitException,         # Rate limit exceeded
)
```

### Error Handling

```python
from moru import Sandbox
from moru.exceptions import (
    AuthenticationException,
    NotFoundException,
    TimeoutException,
    CommandExitException,
    SandboxException,
)

try:
    sandbox = Sandbox.create("my-template")
    result = sandbox.commands.run("python script.py")
except AuthenticationException:
    print("Invalid API key")
except NotFoundException:
    print("Template not found")
except TimeoutException:
    print("Operation timed out")
except CommandExitException as e:
    print(f"Command failed: exit code {e.exit_code}")
    print(f"stderr: {e.stderr}")
except SandboxException as e:
    print(f"Sandbox error: {e}")
finally:
    if sandbox:
        sandbox.kill()
```

---

## Complete Example

```python
from moru import Sandbox, Volume, Template
from moru.template import wait_for_port

# Build custom template
template = (
    Template()
    .from_python_image("3.11")
    .pip_install(["flask", "gunicorn"])
    .set_start_cmd("gunicorn -b 0.0.0.0:8000 app:app", wait_for_port(8000))
)
Template.build(template, alias="flask-app")

# Create volume for persistence
vol = Volume.create(name="flask-workspace")

# Create sandbox with template and volume
with Sandbox.create(
    template="flask-app",
    timeout=3600,
    volume_id=vol.volume_id,
    volume_mount_path="/workspace",
    envs={"FLASK_ENV": "production"},
    metadata={"project": "demo"}
) as sbx:

    # Write app code
    sbx.files.write("/workspace/app.py", """
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from Moru!'
""")

    # Get public URL
    host = sbx.get_host(8000)
    print(f"App running at: {host}")

    # Wait for user
    input("Press Enter to stop...")

# Cleanup: sandbox auto-killed, volume persists
print("Done! Volume data preserved for next session.")
```
