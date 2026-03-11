# Commands Overview

## Synopsis

```
claude-review [command] [flags]
```

Running `claude-review` with no arguments reviews the currently staged changes (equivalent to `claude-review diff --staged`).

## Commands

| Command | Description |
|---------|-------------|
| `claude-review` | Review staged changes (default) |
| `claude-review diff [ref]` | Review a commit range or branch diff |
| `claude-review pr <url\|number>` | Review a GitHub PR, GitLab MR, or Bitbucket PR |
| `claude-review memory <subcommand>` | Manage the persistent memory layer |
| `claude-review insights` | Show cross-PR pattern analysis in plain English |
| `claude-review install-hook` | Install as a pre-commit git hook |

## Global flags

These flags work with every command:

| Flag | Default | Description |
|------|---------|-------------|
| `--format` | `markdown` | Output format: `markdown`, `json`, `annotations` |
| `--output` | `REVIEW.md` | Output file path |
| `--model` | `claude-haiku-4-5-20251001` | Claude model to use |
| `--agents` | `4` | Number of parallel finder agents |
| `--focus` | all | Comma-separated focus areas: `logic,security,performance,types,tests` |
| `--estimate` | `false` | Print cost estimate and exit without running |
| `--verbose` | `false` | Show agent-level debug output |
| `--no-color` | `false` | Disable colored terminal output |
| `--memory` | `false` | Enable memory-augmented review |

## Quick reference

```bash
# Review staged changes
claude-review

# Review last N commits
claude-review diff HEAD~1
claude-review diff HEAD~5

# Review branch against main
claude-review diff main..my-feature

# Review a PR (all platforms)
claude-review pr https://github.com/org/repo/pull/123
claude-review pr https://gitlab.com/org/project/-/merge_requests/45
claude-review pr https://bitbucket.org/workspace/repo/pull-requests/7

# PR shorthand (auto-detects remote)
claude-review pr 123

# CI-friendly JSON output
claude-review pr 123 --format json --output review.json

# GitHub Checks annotations
claude-review pr 123 --format annotations --output annotations.json

# Estimate cost without running
claude-review diff main..feature --estimate

# Security-only review with Sonnet
claude-review pr 123 --focus security --model claude-sonnet-4-6

# Memory-augmented review
claude-review diff --memory

# Install pre-commit hook
claude-review install-hook
```

## Subcommand pages

- [diff →](diff.md)
- [pr →](pr.md)
- [memory →](memory.md)
- [insights →](insights.md)
- [install-hook →](install-hook.md)
