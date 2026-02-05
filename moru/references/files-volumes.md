# Files and Volumes

## Table of Contents
- [Filesystem Overview](#filesystem-overview)
- [Read Files](#read-files)
- [Write Files](#write-files)
- [File Operations](#file-operations)
- [Volumes Overview](#volumes-overview)
- [Create Volume](#create-volume)
- [Attach Volume to Sandbox](#attach-volume-to-sandbox)
- [Volume File Operations](#volume-file-operations)

---

## Filesystem Overview

Each sandbox has an isolated filesystem. Default working directory: `/home/user`

### Quick Reference

| Operation | Python | JavaScript |
|-----------|--------|------------|
| Read | `sbx.files.read(path)` | `sbx.files.read(path)` |
| Write | `sbx.files.write(path, data)` | `sbx.files.write(path, data)` |
| List | `sbx.files.list(path)` | `sbx.files.list(path)` |
| Exists | `sbx.files.exists(path)` | `sbx.files.exists(path)` |
| Info | `sbx.files.get_info(path)` | `sbx.files.getInfo(path)` |
| Delete | `sbx.files.remove(path)` | `sbx.files.remove(path)` |
| Rename | `sbx.files.rename(old, new)` | `sbx.files.rename(old, new)` |
| Mkdir | `sbx.files.make_dir(path)` | `sbx.files.makeDir(path)` |
| Watch | `sbx.files.watch_dir(path)` | `sbx.files.watchDir(path)` |

---

## Read Files

### Read as Text

```python
content = sbx.files.read("/etc/hostname")
content = sbx.files.read("/home/user/script.py", format="text")
```

```typescript
const content = await sbx.files.read("/etc/hostname")
```

### Read as Bytes

```python
data = sbx.files.read("/path/to/image.png", format="bytes")

# Save locally
with open("local_copy.png", "wb") as f:
    f.write(data)
```

```typescript
const bytes = await sbx.files.read("/path/to/file.bin", { format: 'bytes' })
const blob = await sbx.files.read("/path/to/file.bin", { format: 'blob' })
```

### Stream Large Files

```python
stream = sbx.files.read("/path/to/large.bin", format="stream")
with open("local.bin", "wb") as f:
    for chunk in stream:
        f.write(chunk)
```

```typescript
const stream = await sbx.files.read("/path/to/large.bin", { format: 'stream' })
```

### Read as Different User

```python
content = sbx.files.read("/etc/shadow", user="root")
content = sbx.files.read("/home/alice/file.txt", user="alice")
```

---

## Write Files

### Write Text

```python
sbx.files.write("/home/user/hello.txt", "Hello, World!")

# Multi-line
sbx.files.write("/home/user/script.py", """
import sys
print(f"Python {sys.version}")
""")
```

```typescript
await sbx.files.write("/home/user/hello.txt", "Hello, World!")
```

### Write Binary

```python
with open("local_image.png", "rb") as f:
    sbx.files.write("/home/user/image.png", f.read())
```

### Write Multiple Files

```python
files = [
    {"path": "/home/user/file1.txt", "data": "Content 1"},
    {"path": "/home/user/file2.txt", "data": "Content 2"},
    {"path": "/home/user/config.json", "data": '{"key": "value"}'},
]
results = sbx.files.write_files(files)
```

```typescript
await sbx.files.write([
  { path: '/file1.txt', data: 'Content 1' },
  { path: '/file2.txt', data: 'Content 2' },
])
```

### Auto Directory Creation

```python
# Creates /home/user/deep/nested/path/ automatically
sbx.files.write("/home/user/deep/nested/path/file.txt", "content")
```

### Write as Different User

```python
sbx.files.write("/etc/myconfig.conf", "config data", user="root")
```

---

## File Operations

### List Directory

```python
entries = sbx.files.list("/home/user")
for entry in entries:
    if entry.type == "dir":
        print(f"[DIR]  {entry.name}")
    else:
        print(f"[FILE] {entry.name} ({entry.size} bytes)")
```

```typescript
const entries = await sbx.files.list("/home/user")
const recursive = await sbx.files.list("/home/user", { depth: 5 })
```

### Check Existence

```python
if sbx.files.exists("/home/user/file.txt"):
    content = sbx.files.read("/home/user/file.txt")
```

### Get File Info

```python
info = sbx.files.get_info("/home/user/file.txt")
print(f"Name: {info.name}")
print(f"Path: {info.path}")
print(f"Type: {info.type}")  # "file" or "dir"
print(f"Size: {info.size} bytes")
print(f"Mode: {info.mode}")
print(f"Permissions: {info.permissions}")  # e.g., "-rw-r--r--"
print(f"Owner: {info.owner}")
print(f"Group: {info.group}")
print(f"Modified: {info.modified_time}")
```

### Delete File/Directory

```python
sbx.files.remove("/home/user/old_file.txt")
```

### Rename/Move

```python
sbx.files.rename("/old/path/file.txt", "/new/path/file.txt")
```

### Create Directory

```python
sbx.files.make_dir("/home/user/new_dir")
```

### Watch for Changes

```python
handle = sbx.files.watch_dir("/home/user/project")
for event in handle.events():
    print(f"{event.type}: {event.name}")
handle.stop()
```

```typescript
const handle = await sbx.files.watchDir('/path', (event) => {
  console.log(`${event.type}: ${event.name}`)
})
await handle.stop()
```

### Upload/Download URLs

```python
# Get upload URL for direct upload
upload_url = sbx.upload_url("/home/user/large.zip")

# Get download URL
download_url = sbx.download_url("/home/user/output.zip")
```

---

## Volumes Overview

Volumes provide **persistent storage** that survives sandbox restarts.

### When to Use Volumes

- **Agent workspaces** - Git repos, project files between sessions
- **Large datasets** - Data that shouldn't be re-downloaded
- **Shared state** - Files accessed by multiple sandboxes over time
- **Cold access** - Browse/download files without running sandbox

### Volume vs Sandbox Filesystem

| Feature | Volume | Sandbox Filesystem |
|---------|--------|-------------------|
| Persistence | Survives restarts | Lost on kill |
| Access without sandbox | Yes (API) | No |
| Startup time | Instant (lazy load) | Instant |
| Max size | 10 TB | 10 GB |

### Limits

| Limit | Value |
|-------|-------|
| Max volume size | 10 TB |
| Max file size | 1 TB |
| Volume name length | 63 characters |
| Volumes per team | 1000 |

---

## Create Volume

### Python

```python
from moru import Volume

# Create volume
vol = Volume.create(name="my-workspace")
print(f"Volume ID: {vol.volume_id}")
print(f"Name: {vol.name}")
```

### JavaScript

```typescript
import { Volume } from '@moru-ai/core'

const vol = await Volume.create({ name: 'my-workspace' })
```

### Idempotent Creation

```python
# Second create with same name returns existing volume
vol1 = Volume.create(name="shared-data")
vol2 = Volume.create(name="shared-data")
assert vol1.volume_id == vol2.volume_id  # Same!
```

### Naming Rules

- Start with lowercase letter
- Only lowercase letters, numbers, hyphens
- Maximum 63 characters

```python
# Valid
Volume.create(name="my-workspace")
Volume.create(name="agent-data-v2")

# Invalid
Volume.create(name="MyWorkspace")    # No uppercase
Volume.create(name="123-data")       # Must start with letter
```

### Get Volume

```python
vol = Volume.get("vol_abc123")       # By ID
vol = Volume.get("my-workspace")     # By name

print(f"Size: {vol.total_size_bytes} bytes")
print(f"Files: {vol.total_file_count}")
```

### List Volumes

```python
volumes = Volume.list()
for vol in volumes:
    print(f"{vol.name}: {vol.total_size_bytes} bytes")
```

### Delete Volume

```python
vol = Volume.get("my-workspace")
vol.delete()  # WARNING: Permanent!
```

---

## Attach Volume to Sandbox

### Basic Usage

```python
from moru import Sandbox, Volume

vol = Volume.create(name="my-workspace")

sbx = Sandbox.create(
    template="base",
    volume_id=vol.volume_id,
    volume_mount_path="/workspace"
)

# Files in /workspace persist to volume
sbx.commands.run("echo 'Hello' > /workspace/hello.txt")
```

### Valid Mount Paths

Must start with:
- `/workspace/`
- `/data/`
- `/mnt/`
- `/volumes/`

```python
# Valid
Sandbox.create(volume_mount_path="/workspace/data")
Sandbox.create(volume_mount_path="/data/input")
Sandbox.create(volume_mount_path="/mnt/storage")

# Invalid
Sandbox.create(volume_mount_path="/etc/data")      # Wrong prefix
Sandbox.create(volume_mount_path="workspace/data") # Not absolute
```

### Data Persistence Pattern

```python
vol = Volume.create(name="persistent-data")

# First sandbox - write data
sbx1 = Sandbox.create(volume_id=vol.volume_id, volume_mount_path="/workspace")
sbx1.commands.run("git clone https://github.com/user/repo /workspace")
sbx1.kill()

# Second sandbox - data still there
sbx2 = Sandbox.create(volume_id=vol.volume_id, volume_mount_path="/workspace")
result = sbx2.commands.run("ls /workspace")
print(result.stdout)  # Shows cloned repo
```

### Limitations

- **One volume per sandbox** - Only one volume can be attached
- **Mount path fixed** - Cannot change after creation
- **Avoid concurrent writes** - Don't write same files from multiple sandboxes

---

## Volume File Operations

Access volume files **without a running sandbox**.

### List Files

```python
vol = Volume.get("my-workspace")

files = vol.list_files("/")
for f in files:
    if f.type == "directory":
        print(f"[DIR]  {f.name}")
    else:
        print(f"[FILE] {f.name} ({f.size} bytes)")
```

### Download Files

```python
content = vol.download("/README.md")
print(content.decode())

# Save locally
with open("local_readme.md", "wb") as f:
    f.write(vol.download("/README.md"))
```

### Upload Files

```python
vol.upload("/data/input.txt", b"Hello, World!")

# From local file
with open("local_file.txt", "rb") as f:
    vol.upload("/data/input.txt", f.read())
```

### Delete Files

```python
vol.delete("/data/old_file.txt")
vol.delete("/temp", recursive=True)  # Delete directory
```

### Pagination

```python
files, next_token = vol.list_files("/", limit=100)
if next_token:
    more, next_token = vol.list_files("/", limit=100, next_token=next_token)
```

---

## Common Patterns

### Pre-populate Volume Before Sandbox

```python
vol = Volume.create(name="agent-input")
vol.upload("/config.json", b'{"model": "gpt-4"}')
vol.upload("/prompt.txt", b"You are a helpful assistant...")

sbx = Sandbox.create(volume_id=vol.volume_id, volume_mount_path="/input")
sbx.commands.run("cat /input/config.json")  # Config ready
```

### Download Results After Sandbox

```python
vol = Volume.create(name="agent-output")
sbx = Sandbox.create(volume_id=vol.volume_id, volume_mount_path="/output")

sbx.commands.run("python generate_report.py > /output/report.txt")
sbx.kill()

# Download without sandbox
report = vol.download("/output/report.txt")
print(report.decode())
```

### Workspace Persistence Across Sessions

```python
import json

# Save session
vol = Volume.create(name=f"session-{session_id}")
sbx = Sandbox.create(volume_id=vol.volume_id, volume_mount_path="/workspace")
# ... do work ...
sbx.kill()

# Store volume ID
with open("session.json", "w") as f:
    json.dump({"volume_id": vol.volume_id}, f)

# Resume later
with open("session.json") as f:
    data = json.load(f)
vol = Volume.get(data["volume_id"])
sbx = Sandbox.create(volume_id=vol.volume_id, volume_mount_path="/workspace")
# Previous work still there!
```
