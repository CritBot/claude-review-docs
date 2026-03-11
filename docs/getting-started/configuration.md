# Configuration

`claude-review` supports three configuration layers, applied in priority order:

```
CLI flags  >  project config (claude-review.config.json)  >  global config (~/.claude-review.json)  >  defaults
```

## Project config file

Create a `claude-review.config.json` in your project root:

```json
{
  "agents": 4,
  "focus": ["logic", "security", "performance", "types", "tests"],
  "model": "claude-haiku-4-5-20251001",
  "confidence_threshold": 0.80,
  "output": "REVIEW.md",
  "max_tokens_per_agent": 4000,
  "max_cost_usd": 5.00
}
```

This file is read automatically when you run `claude-review` from that directory or any subdirectory.

## Global config file

For preferences that apply across all your projects, create `~/.claude-review.json` with the same structure.

## All options

| Field | Default | CLI flag | Description |
|-------|---------|----------|-------------|
| `model` | `claude-haiku-4-5-20251001` | `--model` | Claude model to use |
| `agents` | `4` | `--agents` | Number of parallel finder agents (1–10) |
| `focus` | all 5 areas | `--focus` | Which concern areas to review |
| `confidence_threshold` | `0.80` | — | Minimum confidence (0–1) to include a finding |
| `output` | `REVIEW.md` | `--output` | Output filename |
| `format` | `markdown` | `--format` | Output format: `markdown`, `json`, `annotations` |
| `max_tokens_per_agent` | `4000` | — | Max output tokens per agent API call |
| `max_cost_usd` | `0` (disabled) | — | Abort if estimated cost exceeds this amount |
| `github_token` | — | — | GitHub API token (also via `GITHUB_TOKEN` env) |
| `gitlab_token` | — | — | GitLab API token (also via `GITLAB_TOKEN` env) |
| `bitbucket_token` | — | — | Bitbucket app password (also via `BITBUCKET_TOKEN` env) |
| `gitlab_base_url` | `https://gitlab.com` | — | Override for self-hosted GitLab instances |
| `memory_db` | `<repo>/.claude-review/memory.db` | — | Path to the SQLite memory database (per-repo by default) |

## Environment variables

All secrets can be set via environment variables (preferred over the config file):

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | **Required.** Your Anthropic API key. |
| `GITHUB_TOKEN` | GitHub personal access token or `GITHUB_TOKEN` from Actions |
| `GITLAB_TOKEN` | GitLab personal access token |
| `BITBUCKET_TOKEN` | Bitbucket app password (`username:password` or token) |
| `AZURE_DEVOPS_TOKEN` | Azure DevOps PAT (v1.3.0) |

## Models

| Model | Speed | Typical cost/review | Best for |
|-------|-------|---------------------|----------|
| `claude-haiku-4-5-20251001` | Fast | ~$0.50–2.00 | Most PRs (default) |
| `claude-sonnet-4-6` | Medium | ~$2–8 | Complex / large PRs |
| `claude-opus-4-6` | Slow | ~$10–30 | Critical security audits |

!!! tip "Cost control"
    Set `max_cost_usd` to a hard limit. If the estimated cost exceeds it, the tool aborts before making any API calls. Combine with `--estimate` to preview cost first.

## Focus areas

| Area | What it reviews |
|------|-----------------|
| `logic` | Off-by-one errors, incorrect conditionals, wrong algorithm implementations, data flow bugs |
| `security` | Injection vulnerabilities, auth/authz flaws, secrets in code, insecure defaults, OWASP Top 10 |
| `performance` | N+1 queries, unnecessary allocations, blocking calls in hot paths, inefficient data structures |
| `types` | Null/nil pointer dereferences, type mismatches, missing error handling, unchecked type assertions |
| `tests` | Missing test coverage for new code, test cases that don't verify failure paths, brittle assertions |

To run only specific areas:

```bash
# Single area
claude-review diff --focus security

# Multiple areas
claude-review diff --focus "logic,security"
```

## Confidence threshold

Each finding includes a `confidence` score (0–1). The default threshold of `0.80` filters out speculative findings. Lower it if you want to see more potential issues; raise it to reduce noise:

```json
{
  "confidence_threshold": 0.70
}
```

## Token limits

`max_tokens_per_agent` controls how verbose each agent's response can be. The default of `4000` is enough for most reviews. For very large diffs, you can raise it:

```json
{
  "max_tokens_per_agent": 8000
}
```

!!! warning
    Higher token limits increase cost proportionally.

## Complete example

A production-ready config for a security-sensitive project:

```json
{
  "agents": 5,
  "focus": ["security", "logic", "types"],
  "model": "claude-sonnet-4-6",
  "confidence_threshold": 0.75,
  "output": "REVIEW.md",
  "max_tokens_per_agent": 6000,
  "max_cost_usd": 10.00
}
```
