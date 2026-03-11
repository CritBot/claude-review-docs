# Using Memory in Reviews

This guide walks through the practical day-to-day usage of the memory layer.

## Setup (one time)

No daemon setup required. The memory layer is zero-configuration: consolidation fires automatically in the background when you run any `claude-review` command and the trigger conditions are met.

Just start using `--memory` on your reviews:

```bash
claude-review diff --memory
claude-review pr 123 --memory
```

The first time you run with `--memory`, the `.claude-review/memory.db` database is created in your repo root. From that point on, the on-wake trigger fires consolidation on your next command invocation once 30 minutes have elapsed or 10+ findings have been stored.

### Optional: verify DB state

```bash
claude-review memory status
```

```
Daemon: not running (not needed — on-wake consolidation is active)
Findings stored: 12 (accepted: 12)
Consolidations:  2
False positives: 0
Last consolidation: 2026-03-11 15:20
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

**Findings not being stored**

Make sure you're running with `--memory` or have it set in config. Without the flag, findings are not stored and the DB is not created.

**Consolidation not firing**

Run any `claude-review` command. If the DB exists and either trigger is met (30 min elapsed or 10+ new findings), you'll see `[memory] consolidating patterns in background...` on stderr.

Check the last consolidation time:
```bash
claude-review memory status
```

**Memory not providing useful context**

This is normal early on. Accumulate more reviews — context improves with volume. Consolidation only fires when there are new findings, so the first few reviews will have minimal context.
