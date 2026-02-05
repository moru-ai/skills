# Building Templates

## Table of Contents
- [Overview](#overview)
- [Default Templates](#default-templates)
- [Template Builder API](#template-builder-api)
- [Base Images](#base-images)
- [Package Installation](#package-installation)
- [File Operations](#file-operations)
- [Startup Commands](#startup-commands)
- [Building Templates](#building-templates)
- [Dockerfile Support](#dockerfile-support)
- [CLI Template Commands](#cli-template-commands)

---

## Overview

Templates define the starting state for sandboxes:
- Base image (OS, language runtime)
- Pre-installed packages
- Files and configuration
- Startup command

**Benefits:**
- **Fast startup**: Sandboxes start in seconds with dependencies pre-installed
- **Consistency**: Every sandbox starts with exact same environment
- **Cost efficiency**: Install once, use many times

---

## Default Templates

| Template | Contents |
|----------|----------|
| `base` | Python 3.11, Node.js 20, Yarn, Git, GitHub CLI |
| `claude-agent-python` | Claude Agent SDK, Python 3.11, Node.js 20, Claude Code CLI |

```python
sandbox = Sandbox.create("base")
sandbox = Sandbox.create("claude-agent-python")
```

---

## Template Builder API

### Python

```python
from moru import Template

template = (
    Template()
    .from_python_image("3.11")
    .pip_install(["requests", "flask"])
    .copy("./app", "/app")
    .set_workdir("/app")
    .set_start_cmd("python app.py")
)

info = Template.build(template, alias="my-flask-app")
print(f"Template ID: {info.template_id}")
```

### JavaScript

```typescript
import { Template } from '@moru-ai/core'

const template = Template()
  .fromPythonImage('3.11')
  .pipInstall(['requests', 'flask'])
  .copy('./app', '/app')
  .setWorkdir('/app')
  .setStartCmd('python app.py')

const info = await Template.build(template, { alias: 'my-flask-app' })
```

---

## Base Images

### Language Images

```python
# Python
Template().from_python_image("3.11")
Template().from_python_image("3.10")
Template().from_python_image()  # Default: Python 3

# Node.js
Template().from_node_image("lts")
Template().from_node_image("20")
Template().from_node_image()  # Default: LTS

# Bun
Template().from_bun_image("latest")
Template().from_bun_image("1.0")
```

### OS Images

```python
Template().from_ubuntu_image("22.04")
Template().from_debian_image("bookworm")
Template().from_base_image()  # Moru's optimized base
```

### Custom Images

```python
# Public Docker Hub
Template().from_image("nginx:alpine")

# Private registry
Template().from_image(
    "myregistry.com/myimage:tag",
    username="user",
    password="pass"
)

# AWS ECR
Template().from_aws_registry(
    "123456789.dkr.ecr.us-east-1.amazonaws.com/myimage:tag",
    credentials={"access_key_id": "...", "secret_access_key": "..."}
)

# From existing Moru template
Template().from_template("my-base-template")
```

---

## Package Installation

### Python (pip)

```python
Template().pip_install(["numpy", "pandas", "scikit-learn"])
Template().pip_install("requests")  # Single package
Template().pip_install(["mypackage"], g=True)  # Global
```

### Node.js (npm)

```python
Template().npm_install(["express", "typescript"])
Template().npm_install(["eslint"], g=True)  # Global
Template().npm_install(["jest"], dev=True)  # Dev dependency
```

### Bun

```python
Template().bun_install(["hono", "drizzle-orm"])
Template().bun_install(["typescript"], g=True)
```

### System (apt)

```python
Template().apt_install(["curl", "git", "build-essential"])
Template().apt_install(["chromium"], no_install_recommends=True)
```

---

## File Operations

### Copy Files

```python
# Single file
Template().copy("./config.json", "/app/config.json")

# Directory
Template().copy("./src", "/app/src")

# Multiple sources
Template().copy(["./src", "./lib"], "/app")

# With permissions
Template().copy("./scripts", "/app/scripts", user="root", mode=0o755)
```

### Create Directories

```python
Template().make_dir("/app/data")
Template().make_dir("/app/logs", mode=0o755)
```

### Run Commands During Build

```python
Template().run_cmd("echo 'Building...'")
Template().run_cmd(["apt-get update", "apt-get upgrade -y"])
Template().run_cmd("npm install", user="node")
```

---

## Startup Commands

### Start Command

```python
# Simple
Template().set_start_cmd("node server.js")

# With ready check (waits for this to succeed)
Template().set_start_cmd(
    "node server.js",
    "curl -s localhost:3000/health"
)
```

### Ready Command Helpers

```python
from moru.template import wait_for_port, wait_for_url, wait_for_file, wait_for_process

# Wait for port to be open
Template().set_start_cmd("python server.py", wait_for_port(8080))

# Wait for URL to respond
Template().set_start_cmd("npm start", wait_for_url("http://localhost:3000/health"))

# Wait for file to exist
Template().set_start_cmd("bash startup.sh", wait_for_file("/tmp/ready"))

# Wait for process to be running
Template().set_start_cmd("service start", wait_for_process("myapp"))
```

### Configuration

```python
Template().set_workdir("/app")
Template().set_user("node")
Template().set_envs({
    "NODE_ENV": "production",
    "PORT": "3000",
})
```

---

## Building Templates

### Synchronous Build

```python
from moru import Template

template = Template().from_python_image().pip_install(["flask"])

info = Template.build(
    template,
    alias="my-template",
    cpu_count=2,
    memory_mb=1024
)

print(f"Template ID: {info.template_id}")
print(f"Build ID: {info.build_id}")
```

### Background Build

```python
info = Template.build_in_background(template, alias="my-template")

# Check status later
status = Template.get_build_status(info)
print(f"Status: {status.status}")  # building, success, failed
```

### Build with Logs

```python
def log_handler(entry):
    print(f"[{entry.level}] {entry.message}")

Template.build(
    template,
    alias="my-template",
    on_build_logs=log_handler
)
```

### Build Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `alias` | string | required | Template name |
| `cpu_count` | number | 2 | Build CPU cores |
| `memory_mb` | number | 1024 | Build memory (MB) |
| `skip_cache` | boolean | false | Skip build cache |
| `on_build_logs` | function | - | Log callback |

---

## Dockerfile Support

### From Dockerfile Content

```python
dockerfile = """
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

CMD ["python", "app.py"]
"""

template = Template().from_dockerfile(dockerfile)
info = Template.build(template, alias="my-python-app")
```

### From Dockerfile Path

```python
template = Template().from_dockerfile("./Dockerfile")
info = Template.build(template, alias="my-app")
```

### Supported Instructions

| Instruction | Support | Notes |
|-------------|---------|-------|
| `FROM` | ✅ Full | Sets base image |
| `RUN` | ✅ Full | Commands during build |
| `COPY` | ✅ Full | Copy files into template |
| `WORKDIR` | ✅ Full | Set working directory |
| `ENV` | ✅ Full | Environment variables |
| `USER` | ✅ Full | Set user context |
| `CMD` | ✅ Full | Converted to start command |
| `ENTRYPOINT` | ⚠️ Partial | Converted to CMD |
| `ADD` | ⚠️ Partial | Local files only, no URLs |
| `ARG` | ❌ Ignored | Use ENV instead |
| `EXPOSE` | ❌ Ignored | All ports accessible |
| `VOLUME` | ❌ Ignored | Use Moru volumes |
| `HEALTHCHECK` | ❌ Ignored | Use ready_cmd |

### Multi-Stage Builds: NOT Supported

If you have multi-stage Dockerfile, either:
1. Build locally, use single-stage production Dockerfile
2. Use Template Builder API instead

### Combine Dockerfile with Builder API

```python
template = (
    Template()
    .from_dockerfile("./Dockerfile")
    .set_envs({"NODE_ENV": "production"})
    .apt_install(["curl"])
    .set_start_cmd("node server.js", "curl -s localhost:3000/health")
)
```

---

## CLI Template Commands

```bash
# Initialize template project
moru template init
moru template init --name my-template --language python-sync

# Build template
moru template create my-template
moru template create my-template --dockerfile ./Dockerfile
moru template create my-template --cmd "python app.py" --ready-cmd "curl localhost:8000"
moru template create my-template --cpu-count 4 --memory-mb 2048

# List templates
moru template list

# Delete template
moru template delete my-template

# Publish (make public)
moru template publish my-template

# Unpublish (make private)
moru template unpublish my-template
```

---

## Complete Example

```python
from moru import Template, Sandbox
from moru.template import wait_for_port

# Define template
template = (
    Template()
    .from_python_image("3.11")
    .apt_install(["curl", "git"])
    .pip_install(["flask", "gunicorn", "requests"])
    .copy("./app", "/app")
    .set_workdir("/app")
    .set_envs({"FLASK_ENV": "production"})
    .set_start_cmd(
        "gunicorn -b 0.0.0.0:8000 app:app",
        wait_for_port(8000)
    )
)

# Build template
info = Template.build(template, alias="my-flask-app")
print(f"Built: {info.template_id}")

# Use template
sandbox = Sandbox.create("my-flask-app")
host = sandbox.get_host(8000)
print(f"App running at: {host}")
```
