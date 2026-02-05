# mill-cache

Fast Mill build cache using APFS clones. Snapshot and restore `out/` directories instantly.

## Why?

Switching branches in a Mill project often means recompiling everything. With large codebases, this wastes time waiting for incremental compilation to catch up.

**mill-cache** solves this by snapshotting your `out/` directory at known-good commits, then restoring them instantly when needed.

## How It Works

### APFS Copy-on-Write

On macOS, APFS supports copy-on-write clones via `cp -c`. Cloning a 5GB directory takes milliseconds because no data is actually copied—just metadata pointers.

```
Snapshot: cp -rc out/ ~/.mill-out-cache/project/abc123/
Restore:  cp -rc ~/.mill-out-cache/project/abc123/ out/
```

Both operations are nearly instant regardless of directory size.

### Instant Restore with Move-Swap

The restore operation uses a move-swap pattern for maximum speed:

```bash
mv out out.old.$$              # instant - just renames
cp -rc "$snapshot" out         # instant - APFS clone
# out.old.$$ left for GC to clean later
```

**Result: 5.8GB restored in ~15ms**

The old `out/` is renamed (instant) and the snapshot is cloned (instant). The old directory is left for the garbage collector to clean up later, keeping the restore operation non-blocking.

### Automatic Ancestor Lookup

When restoring without a specific hash, mill-cache walks your git history to find the nearest ancestor with a cached snapshot:

```bash
mill-cache restore    # finds closest cached ancestor automatically
mill-cache restore abc123   # or specify a commit (partial hash OK)
```

## Usage

```bash
# Create snapshot (requires main branch, clean worktree)
mill-cache snapshot    # or: s

# Restore nearest ancestor snapshot
mill-cache restore     # or: r

# Restore specific snapshot
mill-cache restore abc123

# List snapshots
mill-cache list        # or: ls, l
mill-cache list-all    # or: la

# Show cache stats
mill-cache status      # or: st

# Clean up
mill-cache clean       # current project
mill-cache clean-all   # all projects

# Garbage collect old out/ dirs from restore
mill-cache gc

# Debug mode
DEBUG=1 mill-cache restore
```

## Requirements

- macOS with APFS filesystem
- Git repository with Mill build

## Installation

```bash
cp mill-cache ~/bin/
chmod +x ~/bin/mill-cache
```

### Automatic Garbage Collection (Optional)

Install the launchd agent to run GC hourly:

```bash
cp com.mill-cache.gc.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.mill-cache.gc.plist
```

To check status:
```bash
launchctl list | grep mill-cache
tail -f /tmp/mill-cache-gc.log
```

To uninstall:
```bash
launchctl unload ~/Library/LaunchAgents/com.mill-cache.gc.plist
rm ~/Library/LaunchAgents/com.mill-cache.gc.plist
```

## Configuration

Edit the top of the script:

```bash
CACHE_ROOT="$HOME/.mill-out-cache"  # where snapshots live
MAX_SNAPSHOTS=10                     # per project limit
MAX_SIZE_GB=50                       # per project limit
MAIN_BRANCH="master"                 # branch for snapshots
GC_TARGET_DIR="/path/to/project"    # where GC looks for out.old.* dirs
```

## Cache Structure

```
~/.mill-out-cache/
  <project-name>/
    index.txt           # hash, timestamp, size per line
    <commit-hash>/      # cloned out/ directory
```

## Workflow

1. On main branch with clean worktree, run `mill-cache snapshot`
2. Script syncs upstream, compiles, and snapshots `out/`
3. Later, on any branch, run `mill-cache restore` to get the nearest cached state
4. Mill only recompiles the delta from that snapshot

## Performance

| Operation | Traditional | mill-cache |
|-----------|-------------|------------|
| Delete 5GB out/ | ~5-10s | 0ms (moved to background) |
| Copy 5GB snapshot | ~30-60s | ~15ms (APFS clone) |
| **Total restore** | **~40-70s** | **~15ms** |

The speedup comes from APFS copy-on-write semantics—no actual data is copied until files are modified.
