# Moru Agent Skills

Agent skills for [Moru](https://moru.io) cloud sandboxes - isolated Firecracker microVMs for safe code execution.

## Available Skills

| Skill | Description |
|-------|-------------|
| **moru** | Overview, CLI usage, and core concepts |
| **moru-python** | Python SDK (`pip install moru`) |
| **moru-javascript** | JavaScript/TypeScript SDK (`npm install @moru-ai/core`) |

## Installation

```bash
# Install all skills
npx skills add moru-ai/skills

# Install specific skill
npx skills add moru-ai/skills --skill moru-python
```

## Usage

Skills activate automatically when relevant context is detected:
- Ask about Moru concepts or CLI commands → `moru` skill
- Write Python code with sandboxes → `moru-python` skill
- Write JavaScript/TypeScript code with sandboxes → `moru-javascript` skill

## Quick Start

### Python
```python
from moru import Sandbox

with Sandbox.create() as sbx:
    result = sbx.commands.run("echo 'Hello from Moru!'")
    print(result.stdout)
```

### JavaScript/TypeScript
```typescript
import Sandbox from '@moru-ai/core'

const sbx = await Sandbox.create()
try {
  const result = await sbx.commands.run("echo 'Hello from Moru!'")
  console.log(result.stdout)
} finally {
  await sbx.kill()
}
```

## Resources

- [Documentation](https://docs.moru.io)
- [Dashboard](https://moru.io/dashboard)
- [API Keys](https://moru.io/dashboard?tab=keys)

## Contributing

Skills follow the [Agent Skills](https://agentskills.io) open standard. Each skill directory under `skills/` must include a `SKILL.md` file with YAML frontmatter specifying `name` and `description`.
