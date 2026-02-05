# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mill out/ cache manager - a bash utility that snapshots and restores Mill build output directories using APFS clones. Designed for macOS.

## Key Commands

```bash
# Create snapshot (requires main branch, clean worktree, syncs upstream first)
./mill-cache snapshot    # or: s

# Restore nearest ancestor snapshot
./mill-cache restore     # or: r

# Restore specific snapshot (partial hash OK)
./mill-cache restore abc123

# List snapshots
./mill-cache list        # or: ls, l
./mill-cache list-all    # or: la

# Show cache stats
./mill-cache status      # or: st

# Clean up
./mill-cache clean       # current project
./mill-cache clean-all   # all projects
```

## Architecture

Single bash script (`mill-cache`) with these main functions:

- **snapshot**: Enforces main branch + clean worktree → syncs upstream → runs `./mill __.compile` → copies `out/` to `~/.mill-out-cache/<project>/<full-hash>/`
- **restore**: Finds nearest ancestor snapshot in git history, or restores specific hash (partial match supported)
- **enforce_limits**: Auto-prunes oldest snapshots when count > 10 or size > 50GB per project

Cache structure:
```
~/.mill-out-cache/
  <project-name>/
    index.txt           # hash, timestamp, size per line
    <full-commit-hash>/ # cloned out/ directory
```

## After Updating

Sync changes to user's bin:
```bash
cp mill-cache ~/bin/mill-cache
```

## Configuration (hardcoded at top of script)

- `CACHE_ROOT="$HOME/.mill-out-cache"`
- `MAX_SNAPSHOTS=10` per project
- `MAX_SIZE_GB=50` per project
- `MAIN_BRANCH="master"`
