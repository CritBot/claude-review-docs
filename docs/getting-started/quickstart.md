# Quick Start

This guide walks you through the most common `claude-review` workflows in under five minutes.

## 1. Set your API key

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

## 2. Review staged changes (default)

The simplest use case: stage your changes and run `claude-review`:

```bash
git add src/auth/token.go
claude-review
```

This reviews only the staged diff — exactly what `git diff --cached` would show.

Output is written to `REVIEW.md` in the current directory. Example:

```markdown
# Code Review Report
Branch: feature/auth-refactor → main
Changes: +142 -38 across 8 files

## Summary
| Severity | Count |
|----------|-------|
| 🔴 Critical | 1 |
| 🟠 High | 2 |
| 🟡 Medium | 4 |
| 💡 Suggestion | 1 |

Cost: $0.43 · Agents: 5 finder + verifier + ranker · Time: 38s

---

## 🔴 Critical

### JWT expiry is never validated before granting access

**File**: `src/auth/token.go` · **Line**: 87 · **Confidence**: 95%

The decoded token's expiry (`exp` claim) is extracted but never compared against
the current time. Any expired token will be accepted as valid.

**Suggested Fix**:
if claims.ExpiresAt.Unix() < time.Now().Unix() {
    return nil, ErrTokenExpired
}
```

## 3. Review a specific commit or range

```bash
# Last commit
claude-review diff HEAD~1

# Last 3 commits
claude-review diff HEAD~3

# Branch diff against main
claude-review diff main..feature/auth-refactor

# Specific commit hash
claude-review diff a1b2c3d
```

## 4. Review a GitHub PR

```bash
# Full URL
claude-review pr https://github.com/org/repo/pull/123

# Just the PR number (auto-detects your remote)
claude-review pr 123
```

For PR reviews you need a `GITHUB_TOKEN` set:

```bash
export GITHUB_TOKEN=ghp_...
```

## 5. Review a GitLab MR

```bash
# Full URL
claude-review pr https://gitlab.com/org/project/-/merge_requests/45

# MR number shorthand (auto-detects GitLab remote)
claude-review pr 45
```

Set `GITLAB_TOKEN` before running:

```bash
export GITLAB_TOKEN=glpat-...
```

## 6. Review a Bitbucket PR

```bash
claude-review pr https://bitbucket.org/workspace/repo/pull-requests/7
```

Set `BITBUCKET_TOKEN` (an app password with PR read scope):

```bash
export BITBUCKET_TOKEN=...
```

## 7. Preview cost before running

Don't want to be surprised by the API cost? Use `--estimate`:

```bash
claude-review diff main..feature --estimate
```

This counts tokens and prints the estimated cost without making any API calls.

## 8. Output as JSON (for CI / bots)

```bash
claude-review diff HEAD~1 --format json --output review.json
```

The JSON output is structured for machine consumption — pipe it into Slack bots, dashboard scripts, or GitHub comment automation.

## 9. Focus on a specific concern

```bash
claude-review diff --focus security        # security issues only
claude-review diff --focus logic           # logic bugs only
claude-review diff --focus "security,types"  # multiple
```

## 10. Use a more powerful model for critical PRs

```bash
claude-review diff --model claude-sonnet-4-6    # deeper analysis
claude-review diff --model claude-opus-4-6      # maximum depth (most expensive)
```

## 11. Install as a pre-commit hook

Run reviews automatically before every commit:

```bash
claude-review install-hook
```

Remove it:

```bash
claude-review install-hook --remove
```

## 12. Enable memory across reviews

The memory layer remembers patterns across all your PRs. No setup required — just add `--memory`:

```bash
# Run a review with memory enabled
claude-review diff --memory
claude-review pr 123 --memory
```

After the first review, findings are stored in `.claude-review/memory.db`. On your next command, if 30+ minutes have passed or 10+ new findings exist, consolidation runs automatically in the background — no daemon needed.

```bash
# See what patterns have been detected across your PRs
claude-review insights
```

[Learn more about the memory layer →](../memory/how-it-works.md)

## What's next?

- [All commands reference →](../commands/overview.md)
- [Configuration options →](configuration.md)
- [CI integration guide →](../guides/ci-integration.md)
- [Memory layer deep dive →](../memory/how-it-works.md)
