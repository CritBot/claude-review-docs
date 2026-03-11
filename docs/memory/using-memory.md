# Using Memory in Reviews

This guide walks through the practical day-to-day usage of the memory layer.

## Setup (one time)

### 1. Start the daemon

```bash
claude-review memory start
```

### 2. (Optional) Install as a system service

On macOS:
```bash
claude-review memory install
```

On Linux:
```bash
claude-review memory install
systemctl --user enable claude-review-memory
systemctl --user start claude-review-memory
```

This ensures the daemon starts automatically after reboots.

### 3. Verify it's running

```bash
claude-review memory status
```

```
Memory daemon: running (PID 18423)
Database: ~/.claude-review/memory.db
Findings stored: 0
Consolidations run: 0
```

## Day-to-day usage

### Enable memory for every review

Add `--memory` to your review commands:

```bash
claude-review diff --memory
claude-review pr 123 --memory
```

### Or set it in your config

Add to `~/.claude-review.json` to enable it globally:

```json
{
  "memory": true
}
```

## Building up the knowledge base

Memory becomes more valuable as you accumulate more reviews. The first few reviews with `--memory` won't produce much context (the DB is empty), but after 5–10 reviews the memory starts making a meaningful difference.

**How long until memory is useful?**

| Reviews completed | Memory effect |
|------------------|---------------|
| 1–5 | Minimal — just establishing baseline |
| 5–20 | Memory starts surfacing recurring patterns in the same files |
| 20+ | Consolidation kicks in; cross-PR patterns become visible |
| 50+ | Full value — reliable cross-PR pattern detection, hotspot file awareness |

## Checking what's in memory

```bash
claude-review memory status
```

For a narrative summary of patterns:

```bash
claude-review insights
```

## Interpreting memory-augmented results

When memory is active, the markdown report includes a note at the top:

```markdown
# Code Review Report
*Memory-augmented review — context from 23 past reviews*
```

Individual findings from the memory context may appear as separate notes:

```markdown
### ⚠️ Memory: Recurring pattern
**File**: `src/auth/token.go` has had 3 auth-related findings in the last 30 days.
This finding may be part of a broader pattern — see `claude-review insights` for details.
```

## Team workflows

### Shared team memory

Each developer has their own local `~/.claude-review/memory.db`. There is currently no team-shared memory server — memory is individual.

**Workaround for team sharing**: Run `claude-review insights` periodically and share the output in your team channel or wiki. This gives everyone visibility into patterns without needing a shared server.

### Pre-commit hook + memory

If you've installed the pre-commit hook and have memory enabled:

```bash
claude-review install-hook
```

The hook automatically uses `--memory` if the daemon is running, providing richer context at commit time.

## Resetting memory

To clear all stored data and start fresh:

```bash
claude-review memory clear
```

Good reasons to do this:

- After a major architectural refactor (old findings are no longer relevant)
- When switching to a new project (if using a shared `~/.claude-review.json`)
- If the DB has grown large and you want to reduce noise

## Troubleshooting

**Daemon not starting**

```bash
claude-review memory status
# if "not running", check for stale PID file:
ls ~/.claude-review/
rm ~/.claude-review/memory.pid  # if process is gone but PID file remains
claude-review memory start
```

**Findings not being stored**

Make sure you're running with `--memory` or have it set in config. Without the flag, findings are not stored.

**Memory not providing useful context**

This is normal early on. Accumulate more reviews — context improves with volume.
