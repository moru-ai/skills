---
name: moru
description: "Overview of Moru cloud sandboxes - isolated Firecracker microVMs for safe code execution. Use this skill for general concepts, CLI usage, or when unsure which SDK to use. For SDK-specific guidance, see moru-python or moru-javascript skills."
---

# Moru Cloud Sandboxes

Isolated Firecracker microVMs with dedicated CPU, memory, and filesystem for safe code execution.

## When to Use Which Skill

| Skill | Use When |
|-------|----------|
| **moru** (this) | CLI, general concepts, architecture |
| **moru-python** | Python SDK (`pip install moru`) |
| **moru-javascript** | JavaScript/TypeScript SDK (`npm install @moru-ai/core`) |

## Core Concepts

### Sandbox
Isolated VM with 2 vCPUs, 512MB RAM, 10GB disk. Auto-terminates after 5 minutes (configurable up to 24h).

### Template
Pre-built environment with packages installed. Sandboxes start instantly from templates.

### Volume
Persistent storage that survives sandbox restarts. Mount to `/workspace`, `/data`, `/mnt`, or `/volumes`.

## CLI Quick Start

```bash
# Install
curl -fsSL https://moru.io/cli/install.sh | bash

# Login
moru auth login

# Run command in sandbox
moru sandbox run base echo 'hello world'

# Interactive sandbox
moru sandbox create python

# List/kill
moru sandbox list
moru sandbox kill sbx_abc123
```

## CLI Reference

### Sandbox Commands
```bash
moru sandbox create [template]      # Create interactive sandbox
moru sandbox run [template] <cmd>   # Run command, auto-cleanup
moru sandbox exec <id> <cmd>        # Execute in existing sandbox
moru sandbox list                   # List sandboxes
moru sandbox kill <id>              # Kill sandbox
moru sandbox logs <id> [--follow]   # View logs
moru sandbox metrics <id>           # View CPU/memory/disk
```

### Template Commands
```bash
moru template init                              # Initialize template project
moru template create <name> --dockerfile ./Dockerfile
moru template list
moru template delete <name>
```

### Volume Commands
```bash
moru volume create --name <name>
moru volume list
moru volume delete <name>
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `MORU_API_KEY` | API key (get from https://moru.io/dashboard?tab=keys) |
| `MORU_ACCESS_TOKEN` | Access token from `moru auth login` |

## Resource Limits

| Resource | Default | Maximum |
|----------|---------|---------|
| vCPUs | 2 | Plan-dependent |
| Memory | 512 MB | 4 GB |
| Disk | 10 GB | 10 GB |
| Timeout | 5 min | 1h (hobby) / 24h (pro) |
