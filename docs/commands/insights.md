# insights

Generate a plain-English summary of cross-PR patterns detected by the memory layer.

## Usage

```
claude-review insights
```

## Description

The `insights` command asks Claude to analyze the consolidations stored in the memory database and produce a human-readable summary of recurring issues, hotspot files, and improvement trends across all your recent reviews.

Unlike `memory status` (which shows raw counts), `insights` produces a narrative that is useful to share with your team or include in engineering planning documents.

## Example output

```
Cross-PR Insights (last 30 days · 23 reviews · 847 findings)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

RECURRING PATTERNS

1. Missing nil checks in database layer (7 PRs)
   src/db/users.go, src/db/orders.go, src/db/payments.go are consistently
   flagged for nil pointer dereferences before method calls. Consider adding
   a guard helper or switching to a safer ORM pattern.

2. SQL injection risk in query builders (4 PRs)
   Dynamic WHERE clauses in src/api/filters.go use string concatenation
   instead of parameterized queries. This pattern has appeared in 4 consecutive
   PRs — a refactor is overdue.

3. Goroutine leaks in HTTP handlers (3 PRs)
   Several handlers launch goroutines without context cancellation. When the
   request is cancelled, these goroutines run until process exit.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

HOTSPOT FILES (most frequently flagged)

  src/api/filters.go     — 14 findings across 6 PRs
  src/db/users.go        — 11 findings across 5 PRs
  src/auth/middleware.go  — 9 findings across 4 PRs

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TRENDS

  Security findings: ↑ increasing (2 → 8 this month)
  Performance findings: → stable
  Logic bugs: ↓ decreasing (good progress since adding tests!)
```

## Requirements

The memory daemon must have at least one consolidation run completed. If you haven't started the daemon yet:

```bash
claude-review memory start

# Then run a few reviews with --memory
claude-review diff --memory
claude-review pr 123 --memory

# After the daemon has run consolidation (or at least 20 findings stored):
claude-review insights
```

## Flags

| Flag | Description |
|------|-------------|
| `--output <file>` | Write insights to a file instead of stdout |

## Use cases

- **Weekly engineering sync**: Share a snapshot of recurring issues with your team
- **Tech debt planning**: Identify which files need structural refactoring
- **Onboarding**: Help new engineers understand the codebase's known problem areas
- **Security reviews**: Quickly surface if security findings are trending up
