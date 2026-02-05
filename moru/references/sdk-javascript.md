# JavaScript SDK Reference

## Table of Contents
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Sandbox Class](#sandbox-class)
- [Files Module](#files-module)
- [Commands Module](#commands-module)
- [PTY Module](#pty-module)
- [Template Class](#template-class)
- [Volume Class](#volume-class)
- [Error Classes](#error-classes)

---

## Installation

```bash
npm install @moru-ai/core
```

**Requirements:** Node.js 18+

**TypeScript support included.**

---

## Quick Start

```typescript
import Sandbox from '@moru-ai/core'

// Create sandbox
const sandbox = await Sandbox.create()

// Run command
const result = await sandbox.commands.run("echo 'Hello'")
console.log(result.stdout)

// File operations
await sandbox.files.write("/tmp/test.txt", "Hello")
const content = await sandbox.files.read("/tmp/test.txt")

// Cleanup
await sandbox.kill()
```

---

## Sandbox Class

### Creation

```typescript
import Sandbox from '@moru-ai/core'

// Default
const sandbox = await Sandbox.create()

// With template
const sandbox = await Sandbox.create("python")

// Full options
const sandbox = await Sandbox.create("python", {
  timeoutMs: 600000,                    // milliseconds
  metadata: { key: "value" },
  envs: { VAR: "value" },
  allowInternetAccess: true,
  secure: true,
  volumeId: "vol_abc",
  volumeMountPath: "/workspace",
  network: { allowOut: ["api.example.com"] },
  apiKey: "moru_...",                   // or use MORU_API_KEY env
})
```

### Connection

```typescript
// Connect to existing
const sandbox = await Sandbox.connect("sbx_abc123")

// With options
const sandbox = await Sandbox.connect("sbx_abc123", {
  requestTimeoutMs: 30000
})

// Check status
const isRunning = await sandbox.isRunning()
```

### Properties

```typescript
sandbox.sandboxId           // "sbx_abc123..."
sandbox.sandboxDomain       // Domain
sandbox.trafficAccessToken  // Auth token
```

### Methods

```typescript
// Get info
const info = await sandbox.getInfo()
// Returns: { templateId, startedAt, endAt, state, cpuCount, memoryMb }

// Get metrics
const metrics = await sandbox.getMetrics()
// Returns: array of { cpuUsedPct, memUsed, memTotal, diskUsed, diskTotal }

// Set/extend timeout
await sandbox.setTimeout(1800000)  // milliseconds

// Get host URL for port
const host = sandbox.getHost(8080)  // Returns public URL

// Upload/download URLs
const uploadUrl = sandbox.uploadUrl("/path/to/file")
const downloadUrl = sandbox.downloadUrl("/path/to/file")

// Kill
await sandbox.kill()

// Pause (beta)
await sandbox.betaPause()
```

### Static Methods

```typescript
// List sandboxes (paginated)
const paginator = Sandbox.list()
for await (const info of paginator) {
  console.log(info.sandboxId)
}

// With options
const paginator = Sandbox.list({
  metadata: { project: "myapp" }
})

// Kill by ID
await Sandbox.kill("sbx_abc123")

// Set timeout by ID
await Sandbox.setTimeout("sbx_abc123", 1800000)
```

---

## Files Module

Access via `sandbox.files`

### Read

```typescript
// Text (default)
const content = await sandbox.files.read("/path/to/file.txt")

// Bytes
const bytes = await sandbox.files.read("/path/to/file.bin", { format: 'bytes' })

// Blob
const blob = await sandbox.files.read("/path/to/file.bin", { format: 'blob' })

// Stream
const stream = await sandbox.files.read("/path/to/large.bin", { format: 'stream' })

// As different user
const content = await sandbox.files.read("/etc/shadow", { user: 'root' })
```

### Write

```typescript
// Text
await sandbox.files.write("/path/to/file.txt", "content")

// Multiple files
await sandbox.files.write([
  { path: "/file1.txt", data: "content1" },
  { path: "/file2.txt", data: "content2" },
])

// As different user
await sandbox.files.write("/etc/config", "data", { user: 'root' })
```

### Other Operations

```typescript
// List directory
const entries = await sandbox.files.list("/path/to/dir")
const recursive = await sandbox.files.list("/path/to/dir", { depth: 5 })

// Check existence
const exists = await sandbox.files.exists("/path/to/file")

// Get info
const info = await sandbox.files.getInfo("/path/to/file")
// Returns: { name, path, type, size, mode, permissions, owner, group, modifiedTime }

// Delete
await sandbox.files.remove("/path/to/file")

// Rename/move
await sandbox.files.rename("/old/path", "/new/path")

// Create directory
await sandbox.files.makeDir("/path/to/dir")

// Watch directory
const handle = await sandbox.files.watchDir("/path", (event) => {
  console.log(`${event.type}: ${event.name}`)
})
await handle.stop()
```

---

## Commands Module

Access via `sandbox.commands`

### Run

```typescript
// Basic
const result = await sandbox.commands.run("echo 'Hello'")
// Returns: { stdout, stderr, exitCode }

// With options
const result = await sandbox.commands.run("python script.py", {
  cwd: "/app",
  user: "root",
  envs: { DEBUG: "true" },
  timeoutMs: 120000,
  onStdout: (data) => process.stdout.write(data),
  onStderr: (data) => process.stderr.write(data),
})

// Background
const handle = await sandbox.commands.run("python server.py", { background: true })
await handle.sendStdin("input\n")
const result = await handle.wait()
await handle.kill()
```

### Process Management

```typescript
// List running processes
const processes = await sandbox.commands.list()
for (const proc of processes) {
  console.log(`${proc.pid}: ${proc.command}`)
}

// Kill process
await sandbox.commands.kill(pid)
```

---

## PTY Module

Access via `sandbox.pty`

```typescript
// Create PTY
const handle = await sandbox.pty.create({
  cols: 80,
  rows: 24,
  onData: (data) => process.stdout.write(data)
})
await handle.sendStdin("ls -la\n")
await handle.resize(120, 40)
await handle.kill()

// Start with command
const handle = await sandbox.pty.start("python3", {
  cols: 80,
  rows: 24,
  onData: console.log
})
```

---

## Template Class

```typescript
import { Template } from '@moru-ai/core'

// Build template
const template = Template()
  .fromPythonImage('3.11')
  .pipInstall(['requests', 'flask'])
  .copy('./app', '/app')
  .setWorkdir('/app')
  .setStartCmd('python app.py')

// Synchronous build
const info = await Template.build(template, {
  alias: 'my-template',
  cpuCount: 2,
  memoryMB: 1024,
  onBuildLogs: (entry) => console.log(entry)
})

// Background build
const info = await Template.buildInBackground(template, { alias: 'my-template' })
const status = await Template.getBuildStatus(info)

// Convert to Dockerfile (debugging)
const dockerfile = Template.toDockerfile(template)
```

### Builder Methods

```typescript
// Base images
.fromPythonImage('3.11')
.fromNodeImage('lts')
.fromUbuntuImage('22.04')
.fromBaseImage()
.fromImage('nginx:alpine')
.fromDockerfile('./Dockerfile')
.fromTemplate('existing-template')

// Packages
.pipInstall(['package1', 'package2'])
.npmInstall(['package1'])
.bunInstall(['package1'])
.aptInstall(['package1'])

// Files
.copy('./src', '/app/src')
.makeDir('/app/data')

// Configuration
.runCmd('echo "building"')
.setWorkdir('/app')
.setUser('appuser')
.setEnvs({ NODE_ENV: 'production' })

// Startup
.setStartCmd('python app.py')
.setStartCmd('node server.js', 'curl localhost:3000/health')
```

### Ready Commands

```typescript
import {
  waitForPort,
  waitForURL,
  waitForFile,
  waitForProcess
} from '@moru-ai/core'

.setStartCmd('python app.py', waitForPort(8080))
.setStartCmd('npm start', waitForURL('http://localhost:3000'))
.setStartCmd('bash start.sh', waitForFile('/tmp/ready'))
.setStartCmd('service start', waitForProcess('myapp'))
```

---

## Volume Class

```typescript
import { Volume } from '@moru-ai/core'

// Create
const vol = await Volume.create({ name: 'my-workspace' })

// Get
const vol = await Volume.get('vol_abc123')
const vol = await Volume.get('my-workspace')

// List
for await (const vol of Volume.list()) {
  console.log(`${vol.name}: ${vol.totalSizeBytes} bytes`)
}

// Properties
vol.volumeId
vol.name
vol.totalSizeBytes
vol.totalFileCount
vol.createdAt
vol.updatedAt

// File operations
const files = await vol.listFiles('/')
const { files, nextToken } = await vol.listFiles('/', { limit: 100 })

const content = await vol.download('/path/to/file')
await vol.upload('/path/to/file', Buffer.from('content'))
await vol.delete('/path/to/file')
await vol.delete('/path/to/dir', { recursive: true })

// Delete volume
await vol.delete()
```

---

## Error Classes

```typescript
import {
  SandboxError,              // Base class
  TimeoutError,              // Operation timed out
  NotFoundError,             // Resource not found
  AuthenticationError,       // Invalid API key
  NotEnoughSpaceError,       // Disk full
  CommandExitError,          // Non-zero exit code
  TemplateError,             // Template error
  RateLimitError,            // Rate limit exceeded
  BuildError,                // Build failed
} from '@moru-ai/core'
```

### Error Handling

```typescript
import Sandbox, {
  AuthenticationError,
  NotFoundError,
  TimeoutError,
  CommandExitError,
  SandboxError,
} from '@moru-ai/core'

try {
  const sandbox = await Sandbox.create('my-template')
  const result = await sandbox.commands.run('python script.py')
} catch (error) {
  if (error instanceof AuthenticationError) {
    console.log('Invalid API key')
  } else if (error instanceof NotFoundError) {
    console.log('Template not found')
  } else if (error instanceof TimeoutError) {
    console.log('Operation timed out')
  } else if (error instanceof CommandExitError) {
    console.log(`Command failed: exit code ${error.exitCode}`)
    console.log(`stderr: ${error.stderr}`)
  } else if (error instanceof SandboxError) {
    console.log(`Sandbox error: ${error.message}`)
  }
} finally {
  await sandbox?.kill()
}
```

---

## Complete Example

```typescript
import Sandbox, { Template, Volume, waitForPort } from '@moru-ai/core'

async function main() {
  // Build custom template
  const template = Template()
    .fromPythonImage('3.11')
    .pipInstall(['flask', 'gunicorn'])
    .setStartCmd('gunicorn -b 0.0.0.0:8000 app:app', waitForPort(8000))

  await Template.build(template, { alias: 'flask-app' })

  // Create volume for persistence
  const vol = await Volume.create({ name: 'flask-workspace' })

  // Create sandbox with template and volume
  const sandbox = await Sandbox.create('flask-app', {
    timeoutMs: 3600000,
    volumeId: vol.volumeId,
    volumeMountPath: '/workspace',
    envs: { FLASK_ENV: 'production' },
    metadata: { project: 'demo' }
  })

  try {
    // Write app code
    await sandbox.files.write('/workspace/app.py', `
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from Moru!'
`)

    // Get public URL
    const host = sandbox.getHost(8000)
    console.log(`App running at: ${host}`)

    // Keep running
    await new Promise(resolve => setTimeout(resolve, 60000))
  } finally {
    await sandbox.kill()
  }

  console.log('Done! Volume data preserved for next session.')
}

main().catch(console.error)
```

---

## TypeScript Types

Key types exported from `@moru-ai/core`:

```typescript
interface SandboxOpts {
  timeoutMs?: number
  metadata?: Record<string, string>
  envs?: Record<string, string>
  allowInternetAccess?: boolean
  secure?: boolean
  volumeId?: string
  volumeMountPath?: string
  network?: SandboxNetworkOpts
  apiKey?: string
  requestTimeoutMs?: number
}

interface CommandResult {
  stdout: string
  stderr: string
  exitCode: number
  error?: string
}

interface CommandStartOpts {
  background?: boolean
  cwd?: string
  user?: string
  envs?: Record<string, string>
  timeoutMs?: number
  onStdout?: (data: string) => void | Promise<void>
  onStderr?: (data: string) => void | Promise<void>
  stdin?: boolean
}

interface EntryInfo {
  name: string
  path: string
  type: 'file' | 'dir'
  size: number
  mode: number
  permissions: string
  owner: string
  group: string
  modifiedTime: Date
  symlinkTarget?: string
}
```
