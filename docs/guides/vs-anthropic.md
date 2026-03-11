# claude-review vs Anthropic Code Review

Anthropic offers a built-in Code Review feature as part of their Claude platform. This page explains what it is, how it differs from `claude-review`, and why you might choose one over the other.

## What is Anthropic Code Review?

Anthropic Code Review is a managed service integrated into GitHub. When enabled, it automatically adds a reviewer to your pull requests, posts inline comments, and provides a summary of issues. It is configured through Anthropic's dashboard and requires no self-hosted infrastructure.

As of early 2026, it is available only to GitHub users on Team or Enterprise Anthropic plans.

## Feature comparison

| Feature | Anthropic Code Review | claude-review |
|---|---|---|
| **Cost per review** | $15–25 (managed pricing) | $0.02–3.00 (your API tokens) |
| **Platforms** | GitHub only | GitHub, GitLab, Bitbucket, local git |
| **Plan required** | Team or Enterprise | Any Anthropic API key |
| **Self-hosted** | No — runs on Anthropic's infrastructure | Yes — runs on your machine |
| **Open source** | No | MIT licensed |
| **Source code leaves your system** | Yes — sent to Anthropic's review service | No — sent directly to the Anthropic API (same as any Claude API call) |
| **Memory across PRs** | No | Yes (SQLite memory layer) |
| **Cross-PR pattern detection** | No | Yes (`insights` command) |
| **Pre-commit hook** | No | Yes (`install-hook` command) |
| **Local diff review** | No | Yes (`diff` command) |
| **Cost estimate before running** | No | Yes (`--estimate` flag) |
| **Custom focus areas** | No | Yes (`--focus` flag) |
| **JSON/annotations output** | Limited | Full structured output for CI |
| **Configurable agent count** | No | Yes (1–10 parallel agents) |
| **Works in air-gapped environments** | No | Yes (binary + SQLite, no external deps except Anthropic API) |
| **CI pipeline integration** | GitHub only | Any CI system |
| **Model selection** | Anthropic-controlled | Your choice (Haiku/Sonnet/Opus) |

## When to use Anthropic Code Review

Anthropic Code Review is a good fit if:

- You use GitHub exclusively and want zero setup
- Your team is already on a Team/Enterprise Anthropic plan
- You want inline PR comments posted directly to GitHub without any CLI tooling
- You don't need cross-PR memory or custom configuration

## When to use claude-review

`claude-review` is a better fit if:

- You use GitLab, Bitbucket, or local git repos
- You're on a Starter/individual Anthropic API plan
- Cost is a concern — a typical PR costs 10–50× less
- You want reviews before pushing (local diffs, pre-commit hook)
- You need CI-integrated structured output (JSON, GitHub Checks annotations)
- You want persistent memory and cross-PR pattern detection
- You work in a regulated environment where code cannot transit through external managed services
- You want to customize which concern areas are reviewed, the model, or the review pipeline

## Architecture differences

### Anthropic Code Review
```
GitHub PR  ──►  Anthropic's managed service  ──►  GitHub inline comments
```
The diff is sent to Anthropic's infrastructure, processed by their pipeline, and returned as GitHub PR comments. You have no visibility into or control over the pipeline.

### claude-review
```
Local git diff / GitHub PR / GitLab MR / Bitbucket PR
    │
    ▼
claude-review (running on your machine)
    │
    ├── Query memory DB (if --memory)
    │
    ├── Parallel finder agents (your API key → api.anthropic.com)
    ├── Verifier agent
    ├── Ranker agent
    │
    ├── Write REVIEW.md / review.json / annotations.json
    │
    └── Ingest findings into local memory.db (if --memory)
```

The diff is sent from your machine directly to `api.anthropic.com` — the same endpoint you use for any Claude API call. No additional data handling by an intermediary service.

## Cost breakdown example

For a medium-sized PR (500 changed lines):

**Anthropic Code Review**:
- Fixed fee: $15–25 per review
- Monthly for a 5-engineer team (10 PRs/week): ~$600–1,000/month

**claude-review with Haiku (default)**:
- Typical cost: $0.05–0.30 per review
- Monthly for a 5-engineer team (10 PRs/week): ~$2–12/month

**claude-review with Sonnet (for critical PRs)**:
- Typical cost: $1–3 per review
- Monthly for occasional critical reviews: ~$20–60/month

## Combining both tools

Some teams use both:

- **`claude-review`** runs locally and in CI for every PR (fast, cheap, catches the bulk of issues)
- **Anthropic Code Review** runs for production releases or security-critical branches (managed, posts GitHub comments)

This gives the best of both: continuous coverage at low cost, with managed deep reviews when stakes are highest.

## Summary

`claude-review` was built specifically for teams that:

1. Can't or don't want to use a managed service
2. Need multi-platform support
3. Want lower costs, more control, and persistent cross-PR intelligence

If you're already on GitHub and happy with the managed experience, Anthropic Code Review is convenient. If you need more flexibility, `claude-review` is the open-source alternative.
