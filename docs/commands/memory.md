# memory

Manage the persistent memory layer — a SQLite database that stores findings across all your reviews and surfaces recurring patterns.

## Usage

```
claude-review memory <subcommand>
```

!!! tip "No setup needed for consolidation"
    Consolidation fires automatically in the background on any `claude-review` command invocation once the time or volume trigger is met. The `start`/`stop`/`install` subcommands are optional — useful only for always-on machines where you want consolidation to run even between active uses.

## Subcommands

| Subcommand | Description |
|------------|-------------|
| `status` | Show DB statistics and last consolidation time |
| `clear` | Delete all stored findings from the database |
| `start` | Start the optional background consolidation daemon |
| `stop` | Stop the running daemon |
| `install` | Install the daemon as a system service (launchd on macOS, systemd on Linux) |

## Examples

### Check memory status

```bash
claude-review memory status
```

Output:
```
Daemon: not running (on-wake consolidation is active)
Findings stored: 47 (accepted: 47)
Consolidations:  5
False positives: 2
Last consolidation: 2026-03-11 15:20
```

### Clear all stored data

```bash
claude-review memory clear
```

!!! warning
    This permanently deletes all stored findings, consolidations, and false positive records for the current repo. There is no undo.

### Optional: start the background daemon

For always-on machines (CI servers, developer workstations that never sleep):

```bash
claude-review memory start
```

### Optional: install as a system service

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

### Stop the daemon

```bash
claude-review memory stop
```

## Using memory during reviews

Pass `--memory` to any review command:

```bash
claude-review diff --memory
claude-review pr 123 --memory
```

This does two things:

1. **Before the review**: queries the memory DB for findings in the files being changed, groups them into hotspots, fetches recent consolidated insights, and **injects the context block into every finder agent's prompt**
2. **After the review**: stores all new findings in the DB for future use

Additionally, on every `claude-review` command (with or without `--memory`), the on-wake consolidation check runs and fires a background consolidation if the trigger conditions are met.

[See the Memory Layer deep dive →](../memory/how-it-works.md)
