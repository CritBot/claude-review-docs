# Daemon & Autostart

## On-wake trigger vs daemon — which do you need?

**On-wake trigger (built-in, zero setup)** — consolidation runs in the background at the start of any `claude-review` command if the time or volume trigger is met. This is right for 99% of users:

- Works on any machine, no process to manage
- Pauses when you're not using the tool (closes laptop, goes on holiday)
- Catches up automatically next time you run a review
- No installation, no system permissions

**Background daemon (optional)** — a persistent process that checks every 5 minutes and runs consolidation independently of your usage. Worth it only in specific cases:

| Use case | Right choice |
|----------|-------------|
| Laptop / normal developer machine | On-wake trigger |
| CI pipeline | On-wake trigger (fires during each review run) |
| Shared CI server with infrequent `claude-review` invocations | Daemon |
| Always-on workstation where you want `insights` updated overnight | Daemon |

If you find yourself wondering "should I run `memory start`?" — the answer is almost certainly no. The on-wake trigger handles it.

## What the daemon does

The daemon wakes up every 5 minutes, checks if consolidation should run (30-minute time trigger or 10-finding volume trigger), and if so calls the Consolidation Agent. This agent:

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

| Trigger | Value |
|---------|-------|
| Time elapsed since last consolidation (with ≥1 new finding) | 30 minutes |
| New findings stored since last run | 10 findings |

On the very first run with any findings stored, consolidation runs immediately regardless of elapsed time.

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
