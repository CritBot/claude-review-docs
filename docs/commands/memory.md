# memory

Manage the persistent memory layer — a SQLite database that stores findings across all your reviews and surfaces recurring patterns.

## Usage

```
claude-review memory <subcommand>
```

## Subcommands

| Subcommand | Description |
|------------|-------------|
| `start` | Start the background consolidation daemon |
| `stop` | Stop the running daemon |
| `status` | Show daemon status and database statistics |
| `clear` | Delete all stored findings from the database |
| `install` | Install the daemon as a system service (launchd on macOS, systemd on Linux) |

## Examples

### Start the daemon

```bash
claude-review memory start
```

The daemon runs in the background and periodically consolidates findings from multiple reviews into cross-PR patterns. It runs consolidation every 30 minutes (if new findings exist) or immediately when 10+ new findings have accumulated.

### Check daemon status

```bash
claude-review memory status
```

Output:
```
Memory daemon: running (PID 18423)
Database: ~/.claude-review/memory.db
Findings stored: 847
Consolidations run: 12
Last consolidation: 2 hours ago
```

### Stop the daemon

```bash
claude-review memory stop
```

### Install as a system service

On macOS, installs a launchd plist so the daemon starts automatically on login:

```bash
claude-review memory install
```

On Linux, installs a systemd user service:

```bash
claude-review memory install
systemctl --user enable claude-review-memory
systemctl --user start claude-review-memory
```

### Clear all stored data

```bash
claude-review memory clear
```

!!! warning
    This permanently deletes all stored findings, consolidations, and false positive records. There is no undo.

## Using memory during reviews

To use the memory layer during a review, pass `--memory`:

```bash
claude-review diff --memory
claude-review pr 123 --memory
```

This does two things:

1. **Before the review**: queries the memory DB for recent findings in the same files and injects them as context into the finder agent prompts
2. **After the review**: stores all new findings in the DB for future use

[See the Memory Layer deep dive →](../memory/how-it-works.md)
