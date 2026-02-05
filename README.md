# mill-cache

Fast Mill build cache using APFS clones. Snapshot and restore `out/` directories without copying data.

## Why?

Switching branches in a Mill project often means recompiling everything. With large codebases, this wastes time waiting for incremental compilation to catch up.

**mill-cache** solves this by snapshotting your `out/` directory at known-good commits, then restoring them instantly when needed.

## How It Works

### APFS Copy-on-Write

On macOS, APFS supports copy-on-write clones via `cp -c`. Cloning avoids copying actual file data—only metadata is created.

```
Snapshot: cp -rc out/ ~/.mill-out-cache/project/abc123/
Restore:  cp -rc ~/.mill-out-cache/project/abc123/ out/
```

Clone speed depends on file count, not size. A single large file clones in milliseconds; 268K small files take ~60s due to metadata overhead.

### Instant Restore with Move-Swap

The restore operation uses a move-swap pattern for maximum speed:

```bash
mv out out.old.$$              # ~3ms - just renames
cp -rc "$snapshot" out         # ~60s - APFS clone (metadata creation)
# out.old.$$ left for GC to clean later
```

The old `out/` is renamed instantly, then the snapshot is cloned. The old directory is left for the garbage collector to clean up later, avoiding blocking deletion.

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

Benchmarks on a real Mill project (5.8GB, 268K files, 12K directories):

| Operation | Time |
|-----------|------|
| mv out/ to out.old/ | ~3ms |
| cp -rc (APFS clone) | ~60s |
| **Total restore** | **~60s** |

### cp -rc Breakdown

```
$ time cp -rc snapshot/ out/

real    0m59.342s
user    0m0.420s    ← minimal user-space work
sys     0m49.500s   ← bottleneck: kernel metadata creation
```

84% CPU utilization, almost entirely in kernel creating 268K inodes.

### Why APFS Clone?

We benchmarked several approaches for restoring 268K files:

| Method | Time | Notes |
|--------|------|-------|
| **cp -rc** (APFS clone) | **60s** | Winner - no data copy, just metadata |
| tar extract | 158s | 2.6x slower |
| tar.zst extract | 148s | 2.5x slower |
| ditto | 162s | 2.7x slower |

**The bottleneck is filesystem metadata**, not I/O or CPU. Each of the 268K files requires a new inode and directory entry regardless of method. APFS clone avoids copying actual file data (copy-on-write), making it the fastest option.

Compression doesn't help because:
- System time dominates (~55s in kernel creating metadata)
- User time is minimal (~0.4s)
- Decompression adds CPU overhead without reducing metadata work

### Alternatives Considered

| Approach | Restore Time | Problem |
|----------|--------------|---------|
| Symlinks | ~0ms | Risky: Mill writes corrupt cache |
| Hard links | ~0ms | Same issue as symlinks |
| RAM disk | Fast | Not persistent across reboots |
| Incremental | Varies | Complex to implement correctly |

**Verdict**: APFS clone is optimal for this use case. The ~60s restore time is dominated by unavoidable filesystem operations.

