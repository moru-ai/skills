# Command Execution

## Table of Contents
- [Overview](#overview)
- [Run Commands](#run-commands)
- [Command Options](#command-options)
- [Streaming Output](#streaming-output)
- [Background Commands](#background-commands)
- [Process Management](#process-management)
- [Interactive PTY](#interactive-pty)
- [Error Handling](#error-handling)

---

## Overview

Execute shell commands in sandbox with full control over environment, streaming, and process lifecycle.

### Quick Reference

| Operation | Python | JavaScript |
|-----------|--------|------------|
| Run | `sbx.commands.run(cmd)` | `sbx.commands.run(cmd)` |
| Background | `sbx.commands.run(cmd, background=True)` | `sbx.commands.run(cmd, {background: true})` |
| List | `sbx.commands.list()` | `sbx.commands.list()` |
| Kill | `sbx.commands.kill(pid)` | `sbx.commands.kill(pid)` |
| PTY | `sbx.pty.create()` | `sbx.pty.create()` |

### CommandResult Object

| Property | Type | Description |
|----------|------|-------------|
| `stdout` | string | Standard output |
| `stderr` | string | Standard error |
| `exit_code` / `exitCode` | number | Exit code (0 = success) |
| `error` | string | Error message if failed to execute |

---

## Run Commands

### Basic Execution

```python
result = sbx.commands.run("echo 'Hello'")
print(result.stdout)     # Hello
print(result.exit_code)  # 0
```

```typescript
const result = await sbx.commands.run("echo 'Hello'")
console.log(result.stdout)
console.log(result.exitCode)
```

### Complex Commands

```python
# Pipes
result = sbx.commands.run("cat /etc/passwd | grep root | cut -d: -f1")

# Multi-line script
result = sbx.commands.run("""
for i in 1 2 3; do
    echo "Number: $i"
done
""")

# Shell script
script = """
#!/bin/bash
set -e
cd /home/user
python3 -m venv venv
source venv/bin/activate
pip install requests
"""
result = sbx.commands.run(script, timeout=120)
```

---

## Command Options

### Full Options Reference

| Option (Python) | Option (JS) | Type | Default | Description |
|-----------------|-------------|------|---------|-------------|
| `background` | `background` | boolean | `false` | Run without blocking |
| `envs` | `envs` | object | `{}` | Environment variables |
| `user` | `user` | string | `"user"` | User to run as |
| `cwd` | `cwd` | string | `/home/user` | Working directory |
| `timeout` | `timeoutMs` | number | 60s / 60000ms | Command timeout |
| `on_stdout` | `onStdout` | function | - | Stdout callback |
| `on_stderr` | `onStderr` | function | - | Stderr callback |
| `stdin` | `stdin` | boolean | `false` | Keep stdin open |

### Environment Variables

```python
result = sbx.commands.run(
    "python3 -c 'import os; print(os.environ[\"API_KEY\"])'",
    envs={
        "API_KEY": "secret_123",
        "DEBUG": "true",
    }
)
```

### Working Directory

```python
sbx.files.write("/app/script.py", "print('Hello')")
result = sbx.commands.run("python3 script.py", cwd="/app")
```

### User Context

```python
# Run as root
result = sbx.commands.run("apt-get update", user="root")

# Check user
result = sbx.commands.run("whoami")
print(result.stdout)  # user

result = sbx.commands.run("whoami", user="root")
print(result.stdout)  # root
```

### Timeouts

```python
from moru.exceptions import TimeoutException

try:
    # Command killed after 5 seconds
    result = sbx.commands.run("sleep 60", timeout=5)
except TimeoutException:
    print("Command timed out")

# Long-running with appropriate timeout
result = sbx.commands.run("npm install", timeout=300)
```

```typescript
try {
  await sbx.commands.run("sleep 60", { timeoutMs: 5000 })
} catch (error) {
  if (error instanceof TimeoutError) {
    console.log("Timed out")
  }
}
```

---

## Streaming Output

### Real-time Output

```python
import sys

def handle_stdout(data):
    sys.stdout.write(data)
    sys.stdout.flush()

def handle_stderr(data):
    sys.stderr.write(data)
    sys.stderr.flush()

result = sbx.commands.run(
    "for i in $(seq 1 10); do echo $i; sleep 0.5; done",
    on_stdout=handle_stdout,
    on_stderr=handle_stderr,
    timeout=30
)

# Final result still contains all output
print(f"Total: {len(result.stdout)} chars")
```

```typescript
await sbx.commands.run(
  "for i in 1 2 3; do echo $i; sleep 1; done",
  {
    onStdout: (data) => process.stdout.write(data),
    onStderr: (data) => process.stderr.write(data),
    timeoutMs: 30000
  }
)
```

### Log Processing

```python
def process_log(data):
    for line in data.split('\n'):
        if 'ERROR' in line:
            print(f"[ERROR] {line}")
        elif 'WARN' in line:
            print(f"[WARN] {line}")

result = sbx.commands.run(
    "python3 app.py",
    on_stdout=process_log,
    on_stderr=process_log
)
```

---

## Background Commands

### Start Background Process

```python
handle = sbx.commands.run("python3 server.py", background=True)

# Do other work while server runs...
sbx.commands.run("curl localhost:8000")

# Wait for completion (blocking)
result = handle.wait()

# Or kill it
handle.kill()
```

```typescript
const handle = await sbx.commands.run("python server.py", { background: true })

// Do other work...

// Wait or kill
const result = await handle.wait()
// or
await handle.kill()
```

### Send Input to Process

```python
handle = sbx.commands.run("python3", background=True, stdin=True)
handle.send_stdin("print('Hello')\n")
handle.send_stdin("exit()\n")
result = handle.wait()
```

### Long-running Server Pattern

```python
# Start server in background
handle = sbx.commands.run(
    "python3 -m http.server 8080",
    background=True
)

# Wait for server to be ready
import time
time.sleep(2)

# Use the server
result = sbx.commands.run("curl -s localhost:8080")
print(result.stdout)

# Get public URL
host = sbx.get_host(8080)
print(f"Public URL: {host}")

# Cleanup
handle.kill()
```

---

## Process Management

### List Running Processes

```python
processes = sbx.commands.list()
for proc in processes:
    print(f"PID {proc.pid}: {proc.command}")
```

### Kill Process by PID

```python
sbx.commands.kill(1234)
```

### Graceful Shutdown

```python
# Send SIGTERM first
sbx.commands.run("pkill -SIGTERM python3")
time.sleep(2)

# Then force kill if needed
sbx.commands.run("pkill -SIGKILL python3")
```

---

## Interactive PTY

Create interactive terminal sessions.

### Basic PTY

```python
handle = sbx.pty.create(
    cols=80,
    rows=24,
    on_data=lambda data: print(data.decode(), end="")
)

handle.send_input("ls -la\n")
handle.send_input("pwd\n")
handle.send_input("exit\n")

handle.kill()
```

```typescript
const handle = await sbx.pty.create({
  cols: 80,
  rows: 24,
  onData: (data) => process.stdout.write(data)
})

await handle.sendStdin("ls -la\n")
await handle.resize(120, 40)
await handle.kill()
```

### Start with Command

```python
handle = sbx.pty.start(
    "python3",
    cols=80,
    rows=24,
    on_data=lambda data: print(data.decode(), end="")
)

handle.send_input("print('Hello')\n")
handle.send_input("exit()\n")
handle.kill()
```

### Resize Terminal

```python
handle.resize(120, 40)  # cols, rows
```

---

## Error Handling

### Exit Code Checking

```python
result = sbx.commands.run("grep pattern /nonexistent")
if result.exit_code != 0:
    print(f"Failed with exit code {result.exit_code}")
    print(f"Error: {result.stderr}")
```

### Exception Handling

```python
from moru.exceptions import (
    TimeoutException,
    CommandExitException,
    SandboxException,
)

try:
    result = sbx.commands.run("python3 script.py", timeout=30)
except TimeoutException:
    print("Command timed out")
except CommandExitException as e:
    print(f"Exit code: {e.exit_code}")
    print(f"Stdout: {e.stdout}")
    print(f"Stderr: {e.stderr}")
except SandboxException as e:
    print(f"Sandbox error: {e}")
```

```typescript
import { TimeoutError, CommandExitError, SandboxError } from '@moru-ai/core'

try {
  await sbx.commands.run("python script.py")
} catch (error) {
  if (error instanceof TimeoutError) {
    console.log("Timed out")
  } else if (error instanceof CommandExitError) {
    console.log(`Exit code: ${error.exitCode}`)
    console.log(`Stderr: ${error.stderr}`)
  } else if (error instanceof SandboxError) {
    console.log(`Sandbox error: ${error.message}`)
  }
}
```

---

## Common Patterns

### Install and Run

```python
# Install dependencies
sbx.commands.run("pip install requests flask", user="root", timeout=120)

# Run app
result = sbx.commands.run("python3 app.py", timeout=60)
```

### Git Operations

```python
sbx.commands.run("git clone https://github.com/user/repo /app")
sbx.commands.run("cd /app && git pull")
sbx.commands.run("cd /app && git status")
```

### Build and Test

```python
# Build
result = sbx.commands.run("npm run build", cwd="/app", timeout=300)
if result.exit_code != 0:
    print("Build failed")
    print(result.stderr)

# Test
result = sbx.commands.run("npm test", cwd="/app", timeout=120)
print(f"Tests: {'PASSED' if result.exit_code == 0 else 'FAILED'}")
```

### Service with Health Check

```python
# Start service
handle = sbx.commands.run("node server.js", background=True, cwd="/app")

# Wait for health
import time
for _ in range(30):
    result = sbx.commands.run("curl -s localhost:3000/health")
    if result.exit_code == 0:
        print("Service ready!")
        break
    time.sleep(1)
else:
    print("Service failed to start")
    handle.kill()
```
