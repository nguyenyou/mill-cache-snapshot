---
name: update
description: Install mill-cache scripts to ~/bin. Use when asked to "update", "install", "sync to bin", or "deploy scripts".
---

# Update Scripts to ~/bin

Copy project scripts to user's bin directory.

## Scripts

| Source | Destination |
|--------|-------------|
| `mill-cache` | `~/bin/mill-cache` |
| `mill-daemons` | `~/bin/mill-daemons` |

## Process

1. Ensure `~/bin` exists
2. Copy each script
3. Report results

```bash
mkdir -p ~/bin
cp mill-cache ~/bin/mill-cache
cp mill-daemons ~/bin/mill-daemons
echo "Updated: mill-cache, mill-daemons -> ~/bin/"
```
