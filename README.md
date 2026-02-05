# Moru Agent Skills

Agent skills for [Moru](https://moru.io) cloud sandboxes - isolated Firecracker microVMs for safe code execution.

## Skills Overview

### Essential Skills

| Skill | Description |
|-------|-------------|
| **moru** | CLI commands, core concepts, architecture overview |

### SDK Skills

| Skill | Description |
|-------|-------------|
| **moru-python** | Python SDK (`pip install moru`) - Sandbox, Volume, Template APIs |
| **moru-javascript** | JavaScript/TypeScript SDK (`npm install @moru-ai/core`) - Full TypeScript support |

## Installation

```bash
# Install all skills
npx skills add moru-ai/skills

# Install specific skill
npx skills add moru-ai/skills --skill moru-python
npx skills add moru-ai/skills --skill moru-javascript
```

## Quick Start

### Python

```python
from moru import Sandbox

with Sandbox.create() as sbx:
    sbx.files.write("/app/script.py", "print('Hello from Moru!')")
    result = sbx.commands.run("python3 /app/script.py")
    print(result.stdout)
# Sandbox auto-killed
```

### JavaScript/TypeScript

```typescript
import Sandbox from '@moru-ai/core'

const sbx = await Sandbox.create()
try {
  await sbx.files.write("/app/script.py", "print('Hello from Moru!')")
  const result = await sbx.commands.run("python3 /app/script.py")
  console.log(result.stdout)
} finally {
  await sbx.kill()
}
```

### CLI

```bash
# Install CLI
curl -fsSL https://moru.io/cli/install.sh | bash

# Login
moru auth login

# Run command in sandbox
moru sandbox run base echo 'hello world'

# Create interactive sandbox
moru sandbox create python
```

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Sandbox** | Isolated VM with 2 vCPUs, 1GB RAM (up to 8 vCPUs, 8GB RAM). Default timeout is 1 hour. |
| **Template** | Pre-built environment with packages installed. Sandboxes start instantly from templates. |
| **Volume** | Persistent storage that survives sandbox restarts. Mount to `/workspace`, `/data`, `/mnt`, or `/volumes`. |

## Resources

- [Documentation](https://moru.io/docs)
- [Dashboard](https://moru.io/dashboard)
- [API Keys](https://moru.io/dashboard?tab=keys)

## Contributing

See [AGENTS.md](./AGENTS.md) for contribution guidelines.

Skills follow the [Agent Skills](https://agentskills.io) open standard. Each skill directory under `skills/` must include:

1. `SKILL.md` with YAML frontmatter (`name`, `description`)
2. `LICENSE.txt` with license terms

## License

Apache 2.0 - See [LICENSE.txt](./skills/moru/LICENSE.txt) for details.
