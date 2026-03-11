# Output Formats

`claude-review` supports three output formats controlled by the `--format` flag.

## Markdown (default)

```bash
claude-review diff HEAD~1
# or explicitly:
claude-review diff HEAD~1 --format markdown --output REVIEW.md
```

Produces a human-readable `REVIEW.md` file. This is the default output format.

### Example output

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
```go
if claims.ExpiresAt.Unix() < time.Now().Unix() {
    return nil, ErrTokenExpired
}
```

---

## 🟠 High

### Unvalidated redirect destination
...
```

### When to use Markdown

- Local development — read the report directly
- Committing review artifacts to the repo
- Sharing with teammates via email or Slack

## JSON

```bash
claude-review diff HEAD~1 --format json --output review.json
```

Produces a structured `review.json` file for programmatic consumption.

### Schema

```json
{
  "generated_at": "2026-03-11T14:30:00Z",
  "source": "github-pr",
  "pr_url": "https://github.com/org/repo/pull/123",
  "diff_ref": "main..feature/auth-refactor",
  "summary": {
    "critical": 1,
    "high": 2,
    "medium": 4,
    "suggestion": 1,
    "total": 8
  },
  "findings": [
    {
      "id": "f1a2b3c4",
      "file": "src/auth/token.go",
      "line": 87,
      "severity": "critical",
      "confidence": 0.95,
      "title": "JWT expiry is never validated before granting access",
      "description": "The decoded token's expiry (`exp` claim) is extracted but never compared...",
      "suggested_fix": "if claims.ExpiresAt.Unix() < time.Now().Unix() {\n    return nil, ErrTokenExpired\n}",
      "focus_area": "security",
      "needs_context": false
    }
  ],
  "cost": {
    "input_tokens": 12400,
    "output_tokens": 3200,
    "model": "claude-haiku-4-5-20251001",
    "estimated_usd": 0.43
  },
  "agents_used": 7,
  "duration_seconds": 38
}
```

### When to use JSON

- CI pipelines that post review results as PR comments
- Slack bots that notify teams of critical findings
- Dashboards tracking review quality over time
- Any automated processing of review results

### Example: Post critical findings as a GitHub PR comment

```bash
# In a GitHub Actions workflow
claude-review pr $PR_URL --format json --output review.json

# Extract critical findings count
CRITICAL=$(jq '.summary.critical' review.json)

if [ "$CRITICAL" -gt 0 ]; then
  COMMENT=$(jq -r '.findings[] | select(.severity == "critical") | "- \(.file):\(.line) — \(.title)"' review.json)
  gh pr comment $PR_NUMBER --body "## Critical findings from claude-review\n$COMMENT"
fi
```

## Annotations (GitHub Checks)

```bash
claude-review diff HEAD~1 --format annotations --output annotations.json
```

Produces a `annotations.json` file in GitHub Checks API format for inline PR annotations.

### Schema

```json
[
  {
    "path": "src/auth/token.go",
    "start_line": 87,
    "end_line": 87,
    "annotation_level": "failure",
    "message": "JWT expiry is never validated before granting access",
    "title": "Critical: Security"
  },
  {
    "path": "src/api/handlers.go",
    "start_line": 142,
    "end_line": 142,
    "annotation_level": "warning",
    "message": "Unvalidated redirect destination",
    "title": "High: Security"
  }
]
```

### Annotation levels

| Severity | GitHub annotation level |
|----------|------------------------|
| critical | `failure` |
| high | `failure` |
| medium | `warning` |
| suggestion | `notice` |

### Uploading annotations in GitHub Actions

```yaml
- name: Run claude-review
  run: |
    claude-review pr ${{ github.event.pull_request.html_url }} \
      --format annotations \
      --output annotations.json
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

- name: Upload annotations
  uses: yuzutech/annotations-action@v0.4.0
  if: always()
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    title: Code Review
    input: annotations.json
```

This creates inline annotations directly on the PR diff view in GitHub.

### When to use Annotations

- When you want findings to appear as inline comments on the GitHub PR diff
- CI pipelines that integrate with GitHub Checks
- Teams that prefer GitHub-native review UI

## Choosing a format

| Scenario | Recommended format |
|----------|-------------------|
| Local development | `markdown` |
| CI pipeline, automated comments | `json` |
| GitHub PR inline annotations | `annotations` |
| Sharing review results | `markdown` |
| Dashboard/metrics collection | `json` |
