# Moru CLI Reference

## Table of Contents
- [Installation](#installation)
- [Authentication](#authentication)
- [Sandbox Commands](#sandbox-commands)
- [Template Commands](#template-commands)
- [Volume Commands](#volume-commands)
- [Configuration File](#configuration-file)

---

## Installation

```bash
# Install script (no Node.js required)
curl -fsSL https://moru.io/cli/install.sh | bash

# npm (requires Node.js 20+)
npm install -g @moru-ai/cli

# Homebrew
brew tap moru-ai/moru && brew install moru
```

Verify: `moru --version`

---

## Authentication

```bash
moru auth login              # OAuth browser-based login
moru auth info               # Show current user info
moru auth logout             # Remove credentials
moru auth configure          # Switch teams (interactive)
```

**API Key authentication:**
```bash
export MORU_API_KEY=moru_abc123...
moru sandbox list
```

---

## Sandbox Commands

### Create Sandbox

```bash
moru sandbox create base              # Default 'base' template
moru sandbox create python            # Python template
moru sbx cr python                    # Alias

# With config file
moru sandbox create --config ./moru.toml
```

Creates and connects interactively. Sandbox killed when exiting.

### Run Command (One-shot)

```bash
moru sandbox run base echo 'hello'
moru sandbox run python python3 -c "print('Hello')"
```

Creates sandbox, runs command, shows output, kills sandbox.

### Execute in Running Sandbox

```bash
moru sandbox exec sbx_abc123 'echo hello'
moru sandbox exec sbx_abc123 'python3 script.py'
```

### Connect to Existing

```bash
moru sandbox connect sbx_abc123
moru sbx cn sbx_abc123
```

Sandbox continues running after disconnect.

### List Sandboxes

```bash
moru sandbox list                                    # Running only
moru sandbox list --state running,paused            # Filter by state
moru sandbox list --metadata "project=myapp"        # Filter by metadata
moru sandbox list --format json                     # JSON output
moru sandbox list --limit 10                        # Limit results
moru sbx ls                                         # Alias
```

**Output columns:** Sandbox ID, Template, Alias, Started, Ends, State, vCPUs, RAM MiB

### Kill Sandbox

```bash
moru sandbox kill sbx_abc123                         # Single
moru sandbox kill sbx_abc123 sbx_def456             # Multiple
moru sandbox kill --all                              # All running
moru sandbox kill --all --metadata "env=dev"        # Filter and kill
moru sbx kl sbx_abc123                              # Alias
```

### View Logs

```bash
moru sandbox logs sbx_abc123                         # Recent logs
moru sandbox logs sbx_abc123 --follow               # Stream real-time
moru sandbox logs sbx_abc123 --level WARN           # Filter by level
moru sandbox logs sbx_abc123 --format json          # JSON output
moru sandbox logs sbx_abc123 --loggers "app,system" # Filter by logger
moru sbx lg sbx_abc123                              # Alias
```

**Log levels:** DEBUG, INFO, WARN, ERROR

### View Metrics

```bash
moru sandbox metrics sbx_abc123              # Current metrics
moru sandbox metrics sbx_abc123 --follow     # Stream real-time
moru sandbox metrics sbx_abc123 --format json
moru sbx mt sbx_abc123                       # Alias
```

**Output:**
```
CPU:    15.2% (2 cores)
Memory: 256 MB / 512 MB
Disk:   1.2 GB / 10 GB
```

---

## Template Commands

### Initialize Template Project

```bash
moru template init
moru template init --name my-template --language typescript
moru tpl it

# Languages: typescript, python-sync, python-async
```

### Create/Build Template

```bash
moru template create my-template
moru template create my-template --dockerfile ./Dockerfile
moru template create my-template --cmd "node server.js"
moru template create my-template --cmd "node server.js" --ready-cmd "curl localhost:3000/health"
moru template create my-template --cpu-count 4 --memory-mb 2048
moru template create my-template --no-cache
moru tpl ct my-template
```

**Options:**
| Option | Description |
|--------|-------------|
| `--dockerfile, -d` | Path to Dockerfile |
| `--cmd, -c` | Start command |
| `--ready-cmd` | Health check command |
| `--cpu-count` | Build CPU cores (default: 2) |
| `--memory-mb` | Build memory (default: 512) |
| `--no-cache` | Skip Docker cache |
| `--path` | Root directory path |

### List Templates

```bash
moru template list
moru template list --team team_abc123
moru template list --format json
moru tpl ls
```

**Output columns:** Access, Template ID, Name, vCPUs, RAM, Created by, Created at, Disk

### Delete Template

```bash
moru template delete my-template
moru template delete --config ./moru.toml
moru template delete --select              # Interactive selection
moru template delete my-template --yes     # Skip confirmation
moru tpl dl my-template
```

### Publish Template (Make Public)

```bash
moru template publish my-template
moru template publish --select
moru tpl pb my-template
```

### Unpublish Template (Make Private)

```bash
moru template unpublish my-template
moru template unpublish --select
moru tpl upb my-template
```

---

## Volume Commands

### Create Volume

```bash
moru volume create --name my-workspace
```

### List Volumes

```bash
moru volume list
```

**Output:** NAME, SIZE, FILES, CREATED

### Get Volume Info

```bash
moru volume info my-workspace
moru volume info vol_abc123
```

### List Volume Files

```bash
moru volume files my-workspace
moru volume files my-workspace --path /subdir
```

### Upload Files

```bash
moru volume upload my-workspace --source ./local-file.txt --dest /remote-path.txt
```

### Download Files

```bash
moru volume download my-workspace --path /file.txt --output ./local.txt
```

### Delete Volume

```bash
moru volume delete my-workspace
moru volume delete my-workspace --force    # Skip confirmation
```

**Warning:** Deleting a volume permanently removes all data.

---

## Configuration File

### moru.toml for Sandbox

```toml
template = "my-template"
timeout = 600
metadata = { project = "myapp", environment = "dev" }
```

Use: `moru sandbox create --config ./moru.toml`

### moru.toml for Template

```toml
[template]
id = "tpl_abc123"
name = "my-template"
dockerfile = "./Dockerfile"

[template.build]
cpuCount = 2
memoryMB = 1024
startCmd = "node server.js"
readyCmd = "curl localhost:3000/health"
```

---

## Command Aliases

| Full Command | Alias |
|--------------|-------|
| `moru sandbox` | `moru sbx` |
| `moru sandbox create` | `moru sbx cr` |
| `moru sandbox connect` | `moru sbx cn` |
| `moru sandbox list` | `moru sbx ls` |
| `moru sandbox kill` | `moru sbx kl` |
| `moru sandbox logs` | `moru sbx lg` |
| `moru sandbox metrics` | `moru sbx mt` |
| `moru template` | `moru tpl` |
| `moru template init` | `moru tpl it` |
| `moru template create` | `moru tpl ct` |
| `moru template list` | `moru tpl ls` |
| `moru template delete` | `moru tpl dl` |
| `moru template publish` | `moru tpl pb` |
| `moru template unpublish` | `moru tpl upb` |

---

## Environment Variables

| Variable | Description |
|----------|-------------|
| `MORU_API_KEY` | API key for authentication |
| `MORU_ACCESS_TOKEN` | Access token (from login) |
| `MORU_DOMAIN` | API domain (default: moru.io) |
