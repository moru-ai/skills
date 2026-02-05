# Getting Started with Moru

## Table of Contents
- [Installation](#installation)
- [API Key](#api-key)
- [Authentication](#authentication)
- [Your First Sandbox](#your-first-sandbox)
- [SDK Installation](#sdk-installation)

---

## Installation

### CLI Installation

**Option 1: Install Script (Recommended, no Node.js required)**
```bash
curl -fsSL https://moru.io/cli/install.sh | bash
```

**Option 2: npm (requires Node.js 20+)**
```bash
npm install -g @moru-ai/cli
```

**Option 3: Homebrew (macOS/Linux)**
```bash
brew tap moru-ai/moru && brew install moru
```

**Verify installation:**
```bash
moru --version
```

### Configuration Location
CLI config stored at `~/.moru/config.json`:
```json
{
  "email": "you@example.com",
  "accessToken": "...",
  "teamName": "My Team",
  "teamId": "team_...",
  "teamApiKey": "moru_..."
}
```

---

## API Key

### Getting Your API Key

1. Go to Dashboard: https://moru.io/dashboard?tab=keys
2. Click "Create API Key"
3. Copy and store securely

**Security:** Never expose API key in client-side code or commit to version control.

### Using the API Key

**Environment Variable (Recommended):**
```bash
export MORU_API_KEY=moru_your_api_key_here
```

**In Python:**
```python
from moru import Sandbox

# Uses MORU_API_KEY env var automatically
sandbox = Sandbox.create()

# Or pass directly (not recommended for production)
sandbox = Sandbox.create(api_key="moru_your_api_key_here")
```

**In JavaScript:**
```typescript
import Sandbox from '@moru-ai/core'

// Uses MORU_API_KEY env var automatically
const sandbox = await Sandbox.create()

// Or pass directly
const sandbox = await Sandbox.create({ apiKey: 'moru_your_api_key_here' })
```

---

## Authentication

### CLI Authentication

**Browser-based login:**
```bash
moru auth login
```

**Show current user:**
```bash
moru auth info
```

**Logout:**
```bash
moru auth logout
```

**Switch teams:**
```bash
moru auth configure
```

### CI/CD Authentication

Use API key in CI/CD pipelines:
```bash
MORU_API_KEY=${{ secrets.MORU_API_KEY }} moru sandbox create my-template
```

### Multiple Profiles

```bash
# Production
export MORU_API_KEY=$MORU_PROD_KEY
moru sandbox list

# Development
export MORU_API_KEY=$MORU_DEV_KEY
moru sandbox list
```

---

## Your First Sandbox

### Using CLI

```bash
# Create and connect interactively
moru sandbox create base

# Run a single command
moru sandbox run base echo 'hello world!'

# List running sandboxes
moru sandbox list

# View logs
moru sandbox logs <sandbox-id>

# Kill sandbox
moru sandbox kill <sandbox-id>
```

### Using Python SDK

```python
from moru import Sandbox

# Create sandbox
sandbox = Sandbox.create()
print(f"Sandbox ID: {sandbox.sandbox_id}")

# Run command
result = sandbox.commands.run("echo 'Hello from Moru!'")
print(result.stdout)

# Write and read files
sandbox.files.write("/tmp/test.txt", "Hello, World!")
content = sandbox.files.read("/tmp/test.txt")
print(content)

# Cleanup
sandbox.kill()
```

**With context manager (auto-cleanup):**
```python
from moru import Sandbox

with Sandbox.create() as sandbox:
    result = sandbox.commands.run("python3 --version")
    print(result.stdout)
# Sandbox automatically killed
```

### Using JavaScript SDK

```typescript
import Sandbox from '@moru-ai/core'

// Create sandbox
const sandbox = await Sandbox.create()
console.log(`Sandbox ID: ${sandbox.sandboxId}`)

// Run command
const result = await sandbox.commands.run("echo 'Hello from Moru!'")
console.log(result.stdout)

// Write and read files
await sandbox.files.write("/tmp/test.txt", "Hello, World!")
const content = await sandbox.files.read("/tmp/test.txt")
console.log(content)

// Cleanup
await sandbox.kill()
```

---

## SDK Installation

### Python SDK

```bash
pip install moru
```

**Requirements:** Python 3.8+

**Async support:**
```python
from moru import AsyncSandbox

async def main():
    sandbox = await AsyncSandbox.create()
    result = await sandbox.commands.run("echo 'Hello'")
    await sandbox.kill()
```

### JavaScript SDK

```bash
npm install @moru-ai/core
```

**Requirements:** Node.js 18+

**TypeScript support included.**

---

## Available Templates

| Template | Description |
|----------|-------------|
| `base` | Python 3.11, Node.js 20, Yarn, Git, GitHub CLI |
| `claude-agent-python` | Claude Agent SDK with Python 3.11, Node.js 20, Claude Code CLI |

```python
# Use specific template
sandbox = Sandbox.create("base")
sandbox = Sandbox.create("claude-agent-python")
sandbox = Sandbox.create("my-custom-template")
```

---

## Next Steps

- [Sandbox Operations](sandbox.md) - Create, connect, manage sandboxes
- [CLI Commands](cli.md) - Full CLI reference
- [Templates](templates.md) - Build custom templates
- [Troubleshooting](troubleshooting.md) - Common issues and solutions
