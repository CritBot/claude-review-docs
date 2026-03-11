# claude-review

**Open-source multi-agent code review CLI powered by Claude.**

[![GitHub release](https://img.shields.io/github/v/release/critbot/claude-review)](https://github.com/critbot/claude-review/releases)
[![CI](https://github.com/critbot/claude-review/actions/workflows/ci.yml/badge.svg)](https://github.com/critbot/claude-review/actions/workflows/ci.yml)
[![Coverage](https://img.shields.io/badge/coverage-40%25%2B-brightgreen)](https://github.com/critbot/claude-review/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/critbot/claude-review/blob/main/LICENSE)

---

Self-hosted · Bring-your-own-key · Works on **GitHub, GitLab, Bitbucket, and local git repos**

---

## What is claude-review?

`claude-review` is a command-line tool that reviews your code changes using Anthropic's Claude AI. It runs a **multi-agent pipeline** — several specialized AI agents working in parallel — to surface bugs, security vulnerabilities, performance problems, and type-safety issues in your pull requests or local diffs.

Unlike cloud-hosted review tools, `claude-review` runs entirely on your machine using your own Anthropic API key. Your code is sent directly to the Anthropic API and nowhere else.

## Why claude-review?

Anthropic's built-in Code Review feature is expensive and GitHub-only. `claude-review` fills every gap:

| Feature | Anthropic Code Review | claude-review |
|---|---|---|
| Cost | $15–25/review | ~$0.50–2.00 (your API key) |
| Platforms | GitHub only | GitHub, GitLab, Bitbucket, local |
| Plan required | Team/Enterprise | Any Anthropic API access |
| Self-hosted | No | Yes |
| Open source | No | MIT |
| Memory across PRs | No | Yes (SQLite memory layer) |
| CI integration | Limited | Full JSON/annotations output |

[Learn more about the differences →](guides/vs-anthropic.md)

## How it works

`claude-review` runs a **4-phase multi-agent pipeline**:

```
Phase 1–3 (parallel):   Finder agents
  ├── Logic bug finder
  ├── Security vulnerability finder
  ├── Performance issue finder
  ├── Type safety finder
  └── Test coverage finder

Phase 4a (sequential):  Verifier agent
  └── Deduplicates, filters low-confidence findings

Phase 4b (sequential):  Ranker agent
  └── Sorts by severity, elevates critical-file issues
```

[Deep dive into the pipeline →](guides/pipeline.md)

## Quick example

```bash
export ANTHROPIC_API_KEY=sk-ant-...

# Review staged changes
git add .
claude-review

# Review a GitHub PR
claude-review pr https://github.com/org/repo/pull/123

# Review by PR number (auto-detects remote)
claude-review pr 123
```

[Full quickstart guide →](getting-started/quickstart.md)

## Key features

- **Multi-platform**: GitHub, GitLab, Bitbucket, local `git diff`
- **Multiple output formats**: Markdown report, JSON for CI, GitHub Checks annotations
- **Memory layer**: SQLite-backed persistent findings across PRs — surfaces recurring bugs and cross-PR patterns
- **Cost transparency**: Shows exact API cost per review; `--estimate` flag before committing to a run
- **Pre-commit hook**: `claude-review install-hook` catches issues before they land
- **CI-ready**: Structured JSON output for bots, Slack notifications, dashboards

## Installation

```bash
# macOS
brew install critbot/tap/claude-review

# Linux
curl -sSL https://github.com/critbot/claude-review/releases/latest/download/claude-review-linux-amd64.tar.gz | tar xz
sudo mv claude-review /usr/local/bin/
```

[Full installation guide →](getting-started/installation.md)
