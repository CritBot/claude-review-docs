# Daemon & Autostart

The consolidation daemon is a lightweight background process that periodically analyzes stored findings and produces cross-PR pattern summaries.

## What the daemon does

The daemon wakes up every 6 hours (or when 20+ new findings have been stored since the last run) and calls the Consolidation Agent. This agent:

1. Reads all findings stored since the last consolidation
2. Groups them by file, severity, and description similarity
3. Calls the Anthropic API with **metadata only** (no source code) to identify patterns
4. Writes structured consolidation records to the `consolidations` table

The consolidation results power the `claude-review insights` command and the memory context injected into future reviews.

## Daemon commands

```bash
claude-review memory start    # start the daemon
claude-review memory stop     # stop the daemon
claude-review memory status   # show running status and stats
```

## PID file

The daemon writes its PID to `~/.claude-review/memory.pid`. If the process dies unexpectedly and leaves a stale PID file:

```bash
rm ~/.claude-review/memory.pid
claude-review memory start
```

## Autostart on macOS (launchd)

```bash
claude-review memory install
```

This creates a launchd plist at `~/Library/LaunchAgents/com.critbot.claude-review-memory.plist` and loads it immediately.

To verify:

```bash
launchctl list | grep claude-review
```

To unload manually:

```bash
launchctl unload ~/Library/LaunchAgents/com.critbot.claude-review-memory.plist
```

## Autostart on Linux (systemd)

```bash
claude-review memory install
```

This creates a systemd user service at `~/.config/systemd/user/claude-review-memory.service`.

To enable and start:

```bash
systemctl --user daemon-reload
systemctl --user enable claude-review-memory
systemctl --user start claude-review-memory
```

To check status:

```bash
systemctl --user status claude-review-memory
journalctl --user -u claude-review-memory -f
```

## Consolidation triggers

The daemon consolidates when **either** condition is met:

| Trigger | Default |
|---------|---------|
| Time elapsed since last consolidation | 6 hours |
| New findings stored since last run | 20 findings |

These are not currently configurable, but tuning them will be added in a future release.

## Resource usage

The daemon is extremely lightweight:

- **Memory**: ~5–10 MB RSS
- **CPU**: Negligible except during consolidation (a few seconds every 6 hours)
- **Network**: Only during consolidation — sends a small metadata payload to the Anthropic API
- **Disk**: SQLite DB grows at ~50–200 bytes per finding; a year of active usage is typically under 5 MB

## Running without the daemon

The daemon is optional. Without it:

- `--memory` flag still stores and retrieves findings
- No consolidation runs automatically
- `claude-review insights` returns limited results (no cross-PR patterns)

To manually trigger a consolidation:

```bash
# Not yet exposed as a direct command — start the daemon to trigger consolidation,
# then stop it once it completes
claude-review memory start
# Wait ~30 seconds for first consolidation
claude-review memory stop
```

Manual consolidation triggering will be a proper command in a future release.
