# Troubleshooting Guide

## Table of Contents
- [Common Errors](#common-errors)
- [Authentication Issues](#authentication-issues)
- [Sandbox Issues](#sandbox-issues)
- [Command Execution Issues](#command-execution-issues)
- [File Operation Issues](#file-operation-issues)
- [Template Build Issues](#template-build-issues)
- [Volume Issues](#volume-issues)
- [Network Issues](#network-issues)
- [Resource Limits](#resource-limits)
- [Debugging Tips](#debugging-tips)

---

## Common Errors

### Error Types Reference

| Error | Cause | Solution |
|-------|-------|----------|
| `AuthenticationError` | Invalid or missing API key | Check MORU_API_KEY env var |
| `NotFoundError` | Resource doesn't exist | Verify sandbox/template ID |
| `TimeoutError` | Operation took too long | Increase timeout value |
| `NotEnoughSpaceError` | Disk full | Delete files or use larger disk |
| `CommandExitError` | Command failed (non-zero exit) | Check stderr for details |
| `RateLimitError` | Too many requests | Implement backoff/retry |
| `TemplateError` | Template incompatibility | Rebuild template |
| `BuildError` | Template build failed | Check build logs |

---

## Authentication Issues

### "Invalid API key"

**Symptoms:** `AuthenticationError` or 401 responses

**Solutions:**

1. **Check environment variable:**
   ```bash
   echo $MORU_API_KEY
   ```

2. **Verify API key format:**
   ```
   moru_abc123...  # Should start with "moru_"
   ```

3. **Check key permissions in dashboard:**
   https://moru.io/dashboard?tab=keys

4. **Try explicit API key:**
   ```python
   sandbox = Sandbox.create(api_key="moru_your_key_here")
   ```

### "Access token expired"

**Symptoms:** CLI commands fail after working previously

**Solution:**
```bash
moru auth logout
moru auth login
```

### Team/Organization Issues

```bash
# Check current team
moru auth info

# Switch teams
moru auth configure
```

---

## Sandbox Issues

### "Sandbox not found"

**Symptoms:** `NotFoundError` when connecting

**Causes:**
- Sandbox was killed or timed out
- Invalid sandbox ID
- Sandbox belongs to different team

**Solutions:**

1. **List active sandboxes:**
   ```python
   for info in Sandbox.list():
       print(f"{info.sandbox_id}: {info.state}")
   ```

2. **Check if ID is correct:**
   ```python
   # IDs start with "sbx_"
   sandbox = Sandbox.connect("sbx_abc123...")
   ```

### "Sandbox not responding"

**Solutions:**

1. **Check if running:**
   ```python
   if sandbox.is_running():
       print("Still running")
   else:
       print("Terminated - create new sandbox")
   ```

2. **View logs:**
   ```bash
   moru sandbox logs sbx_abc123
   ```

3. **Check metrics:**
   ```bash
   moru sandbox metrics sbx_abc123
   ```

### "Sandbox creation timed out"

**Causes:**
- Template build in progress
- Network issues
- High load

**Solutions:**

1. **Retry with longer timeout:**
   ```python
   sandbox = Sandbox.create(request_timeout=120)
   ```

2. **Use pre-built template:**
   ```python
   # Use default templates for faster startup
   sandbox = Sandbox.create("base")
   ```

### Sandbox Terminated Unexpectedly

**Check remaining time:**
```python
info = sandbox.get_info()
print(f"Ends at: {info.end_at}")
```

**Extend timeout:**
```python
sandbox.set_timeout(3600)  # 1 hour
```

**Keep-alive pattern:**
```python
import threading, time

def keep_alive():
    while True:
        try:
            sandbox.set_timeout(600)
            time.sleep(300)
        except:
            break

threading.Thread(target=keep_alive, daemon=True).start()
```

---

## Command Execution Issues

### "Command timed out"

**Default timeout:** 60 seconds

**Solution:**
```python
# Increase command timeout
result = sandbox.commands.run("npm install", timeout=300)
```

### Non-zero Exit Code

```python
result = sandbox.commands.run("python script.py")

if result.exit_code != 0:
    print(f"Exit code: {result.exit_code}")
    print(f"Stderr: {result.stderr}")
    print(f"Stdout: {result.stdout}")
```

### Command Not Found

**Check if binary exists:**
```python
result = sandbox.commands.run("which python3")
if result.exit_code != 0:
    # Install it
    sandbox.commands.run("apt-get update && apt-get install -y python3", user="root")
```

### Permission Denied

**Run as root:**
```python
sandbox.commands.run("apt-get update", user="root")
```

### Background Process Not Starting

```python
# Verify process is running
handle = sandbox.commands.run("python server.py", background=True)

import time
time.sleep(2)

processes = sandbox.commands.list()
for p in processes:
    print(f"{p.pid}: {p.command}")
```

---

## File Operation Issues

### "File not found"

```python
# Check if file exists first
if sandbox.files.exists("/path/to/file"):
    content = sandbox.files.read("/path/to/file")
else:
    print("File does not exist")
```

### "Permission denied" on files

```python
# Read/write as root
content = sandbox.files.read("/etc/shadow", user="root")
sandbox.files.write("/etc/config", "data", user="root")
```

### "Not enough space"

```python
from moru.exceptions import NotEnoughSpaceException

try:
    sandbox.files.write("/large_file", data)
except NotEnoughSpaceException:
    # Clean up
    sandbox.commands.run("rm -rf /tmp/* ~/.cache/*")
    # Retry
    sandbox.files.write("/large_file", data)
```

### Large File Handling

```python
# Use streaming for large files
stream = sandbox.files.read("/large.bin", format="stream")
with open("local.bin", "wb") as f:
    for chunk in stream:
        f.write(chunk)
```

---

## Template Build Issues

### "Build failed"

**View build logs:**
```python
def log_handler(entry):
    print(f"[{entry.level}] {entry.message}")

Template.build(template, alias="my-template", on_build_logs=log_handler)
```

**CLI:**
```bash
moru template create my-template --dockerfile ./Dockerfile
# Watch output for errors
```

### Common Build Errors

**Missing dependencies:**
```python
template = (
    Template()
    .from_ubuntu_image("22.04")
    .apt_install(["curl", "git", "build-essential"])  # Add missing deps
    .pip_install(["numpy"])
)
```

**Network issues during build:**
```python
# Retry with fresh cache
Template.build(template, alias="my-template", skip_cache=True)
```

### Multi-stage Dockerfile Not Supported

**Error:** Build fails with multi-stage Dockerfile

**Solution:** Use single-stage or Template Builder API
```python
# Instead of multi-stage, build locally and use single stage
template = (
    Template()
    .from_image("node:20-slim")
    .copy("./dist", "/app/dist")
    .set_start_cmd("node /app/dist/server.js")
)
```

---

## Volume Issues

### "Volume not found"

```python
# List volumes to find correct ID/name
for vol in Volume.list():
    print(f"{vol.name}: {vol.volume_id}")

# Get by name or ID
vol = Volume.get("my-workspace")  # by name
vol = Volume.get("vol_abc123")    # by ID
```

### Invalid Mount Path

**Error:** Invalid volume mount path

**Valid prefixes:**
- `/workspace/`
- `/data/`
- `/mnt/`
- `/volumes/`

```python
# Correct
Sandbox.create(volume_mount_path="/workspace/data")

# Wrong
Sandbox.create(volume_mount_path="/home/user/data")  # Invalid prefix
```

### Data Not Persisting

**Ensure writing to mount path:**
```python
vol = Volume.create(name="test")
sbx = Sandbox.create(volume_id=vol.volume_id, volume_mount_path="/workspace")

# This persists (in /workspace)
sbx.files.write("/workspace/data.txt", "persisted!")

# This does NOT persist (outside mount)
sbx.files.write("/home/user/data.txt", "lost on kill!")
```

---

## Network Issues

### "Connection refused"

**Check sandbox allows internet:**
```python
sandbox = Sandbox.create(allow_internet_access=True)
```

**Check if service is running:**
```python
result = sandbox.commands.run("curl -s localhost:8080")
if result.exit_code != 0:
    print("Service not running")
```

### External Access Not Working

**Get public URL:**
```python
# Start server
sandbox.commands.run("python -m http.server 8080", background=True)

# Get public URL
host = sandbox.get_host(8080)
print(f"Access at: {host}")
```

### Network Restrictions

```python
# Allow only specific destinations
sandbox = Sandbox.create(
    network={
        "allow_out": ["api.openai.com", "github.com"],
        "deny_out": ["*"],
    }
)
```

---

## Resource Limits

### Default Limits

| Resource | Default | Maximum |
|----------|---------|---------|
| vCPUs | 2 | Plan-dependent |
| Memory | 512 MB | 4 GB |
| Disk | 10 GB | 10 GB |
| Timeout | 5 min | 1h (hobby) / 24h (pro) |
| Concurrent sandboxes | 20 | Plan-dependent |

### Quota Exceeded

```python
from moru.exceptions import SandboxException

try:
    sandbox = Sandbox.create()
except SandboxException as e:
    if "quota" in str(e).lower():
        # Kill old sandboxes
        for info in Sandbox.list():
            Sandbox.kill(info.sandbox_id)
        # Retry
        sandbox = Sandbox.create()
```

### Monitor Resources

```python
metrics = sandbox.get_metrics()
for m in metrics:
    print(f"CPU: {m.cpu_used_pct:.1f}%")
    print(f"Memory: {m.mem_used / 1024 / 1024:.0f} / {m.mem_total / 1024 / 1024:.0f} MB")
    print(f"Disk: {m.disk_used / 1024 / 1024:.0f} / {m.disk_total / 1024 / 1024:.0f} MB")
```

**CLI:**
```bash
moru sandbox metrics sbx_abc123 --follow
```

---

## Debugging Tips

### Enable Verbose Logging

**CLI:**
```bash
MORU_DEBUG=true moru sandbox create base
```

**SDK:**
```python
import logging
logging.basicConfig(level=logging.DEBUG)

sandbox = Sandbox.create()
```

### View Sandbox Logs

```bash
# Recent logs
moru sandbox logs sbx_abc123

# Stream in real-time
moru sandbox logs sbx_abc123 --follow

# Filter by level
moru sandbox logs sbx_abc123 --level ERROR
```

### Debug Commands

```python
# Print full command output
result = sandbox.commands.run("your_command")
print(f"Exit code: {result.exit_code}")
print(f"Stdout:\n{result.stdout}")
print(f"Stderr:\n{result.stderr}")
```

### Check Sandbox State

```python
info = sandbox.get_info()
print(f"State: {info.state}")
print(f"Template: {info.template_id}")
print(f"Started: {info.started_at}")
print(f"Ends: {info.end_at}")
print(f"CPUs: {info.cpu_count}")
print(f"Memory: {info.memory_mb} MB")
```

### Test Connectivity

```python
# Test internet
result = sandbox.commands.run("curl -s https://httpbin.org/ip")
print(result.stdout)

# Test DNS
result = sandbox.commands.run("nslookup google.com")
print(result.stdout)
```

### Common Debug Commands

```bash
# In sandbox
whoami              # Check user
pwd                 # Check directory
env                 # Check environment
df -h               # Check disk space
free -m             # Check memory
ps aux              # List processes
netstat -tlnp       # Check listening ports
```

---

## Getting Help

- **Documentation:** https://docs.moru.io
- **Dashboard:** https://moru.io/dashboard
- **Support:** hi@moru.io
- **Status:** https://status.moru.io
