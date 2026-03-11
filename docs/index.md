# claude-review

**Open-source multi-agent code review CLI powered by Claude — with a memory that learns your codebase.**

[![GitHub release](https://img.shields.io/github/v/release/critbot/claude-review)](https://github.com/critbot/claude-review/releases)
[![CI](https://github.com/critbot/claude-review/actions/workflows/ci.yml/badge.svg)](https://github.com/critbot/claude-review/actions/workflows/ci.yml)
[![Coverage](https://img.shields.io/badge/coverage-40%25%2B-brightgreen)](https://github.com/critbot/claude-review/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/critbot/claude-review/blob/main/LICENSE)

---

Self-hosted · Bring-your-own-key · Works on **GitHub, GitLab, Bitbucket, and local git repos**

---

## What makes this different

Most AI code review tools — including Anthropic's own $25/review managed service — treat every PR in isolation. They look at the diff, produce findings, and forget everything. The next PR starts from zero.

`claude-review` doesn't.

After every review, findings are stored in a local SQLite database. A background daemon wakes every 30 minutes and uses Claude to find patterns across your entire review history — patterns a single-PR review can never surface. Before the next review starts, those patterns are injected into every finder agent's prompt. The tool already knows which files in your repo are hotspots and which categories of bugs keep slipping through.

**After a month of use, your instance of `claude-review` knows things about your codebase that Anthropic's managed service never will — because theirs resets to zero on every PR.**

The memory is entirely local, stored in `.claude-review/memory.db` in your repo root. No cloud sync. No vector database. No embeddings.

[Deep dive into the memory layer →](memory/how-it-works.md)

---

## What is claude-review?

`claude-review` is a command-line tool that reviews your code changes using Anthropic's Claude AI. It runs a **multi-agent pipeline** — several specialized AI agents working in parallel — to surface bugs, security vulnerabilities, performance problems, and type-safety issues in your pull requests or local diffs.

Unlike cloud-hosted review tools, `claude-review` runs entirely on your machine using your own Anthropic API key. Your code is sent directly to the Anthropic API and nowhere else.

## How it compares

| Feature | Anthropic Code Review | claude-review |
|---|---|---|
| Cost | $15–25/review | ~$0.02–3.00 (your API tokens) |
| Platforms | GitHub only | GitHub, GitLab, Bitbucket, local |
| Plan required | Team or Enterprise | Any Anthropic API key |
| Self-hosted | No | Yes |
| Open source | No | MIT licensed |
| **Memory across PRs** | **No — resets every PR** | **Yes — learns over time** |
| Cross-PR pattern detection | No | Yes (`insights` command) |
| False positive suppression | No | Yes — remembered across PRs |
| Pre-commit hook | No | Yes |
| Local diff review | No | Yes |
| Cost estimate before running | No | Yes (`--estimate` flag) |
| Custom focus areas | No | Yes (`--focus` flag) |
| CI pipeline integration | GitHub only | Any CI system |
| Model selection | Anthropic-controlled | Your choice (Haiku/Sonnet/Opus) |

[Full comparison →](guides/vs-anthropic.md)

---

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

---

## The always-on memory agent

Three jobs run as a persistent local process:

**Ingest** — after every review run with `--memory`, all findings go into `.claude-review/memory.db`: file, line, severity, category, whether you rejected them as noise. Automatic.

**Consolidate** — the daemon wakes every 30 minutes (or after 10 new findings) and calls Claude with *metadata only — no source code* — to find cross-PR patterns. "Your payments module has had null pointer bugs in 4 of the last 6 PRs." Costs fractions of a cent per cycle.

**Query** — before the next review, finder agents query memory for the files being changed. They already know which areas are hotspots and which findings you've previously rejected.

```bash
claude-review memory start      # start the daemon
claude-review diff --memory     # review with memory context
claude-review insights          # plain-English cross-PR pattern summary
```

[Memory layer documentation →](memory/how-it-works.md)

---

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

---

## Installation

```bash
# macOS
brew install critbot/tap/claude-review

# Linux
curl -sSL https://github.com/critbot/claude-review/releases/latest/download/claude-review-linux-amd64.tar.gz | tar xz
sudo mv claude-review /usr/local/bin/
```

[Full installation guide →](getting-started/installation.md)
