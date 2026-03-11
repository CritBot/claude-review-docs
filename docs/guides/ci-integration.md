# CI Integration

`claude-review` is designed to fit naturally into CI pipelines. This page covers integration patterns for GitHub Actions, GitLab CI, and other CI systems.

## GitHub Actions

### Basic PR review

```yaml
# .github/workflows/review.yml
name: Code Review

on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install claude-review
        run: |
          curl -sSL https://github.com/critbot/claude-review/releases/latest/download/claude-review-linux-amd64.tar.gz | tar xz
          sudo mv claude-review /usr/local/bin/

      - name: Review PR
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          claude-review pr ${{ github.event.pull_request.html_url }} \
            --format json \
            --output review.json

      - name: Upload review artifact
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: code-review
          path: review.json
```

### Block PRs on critical findings

```yaml
- name: Review PR and enforce quality gate
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    claude-review pr ${{ github.event.pull_request.html_url }} \
      --format json \
      --output review.json

    CRITICAL=$(jq '.summary.critical' review.json)
    HIGH=$(jq '.summary.high' review.json)

    if [ "$CRITICAL" -gt 0 ]; then
      echo "❌ Found $CRITICAL critical issue(s). Review REVIEW.md for details."
      exit 1
    fi

    if [ "$HIGH" -gt 3 ]; then
      echo "⚠️ Found $HIGH high-severity issues (threshold: 3). Review required."
      exit 1
    fi

    echo "✅ Review passed"
```

### Post findings as PR comments

```yaml
- name: Review PR
  id: review
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    claude-review pr ${{ github.event.pull_request.html_url }} \
      --format json \
      --output review.json

- name: Post review comment
  if: always()
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    CRITICAL=$(jq '.summary.critical' review.json)
    HIGH=$(jq '.summary.high' review.json)
    MEDIUM=$(jq '.summary.medium' review.json)

    # Build comment body
    BODY=$(cat <<EOF
    ## 🤖 claude-review Results

    | Severity | Count |
    |----------|-------|
    | 🔴 Critical | $CRITICAL |
    | 🟠 High | $HIGH |
    | 🟡 Medium | $MEDIUM |

    $(jq -r '.findings[] | select(.severity == "critical" or .severity == "high") | "### \(.severity | ascii_upcase): \(.title)\n**\(.file):\(.line)** — \(.description)\n"' review.json)

    *Cost: $(jq -r '.cost.estimated_usd' review.json | awk '{printf "$%.3f", $1}') · $(jq '.agents_used' review.json) agents · $(jq '.duration_seconds' review.json)s*
    EOF
    )

    gh pr comment ${{ github.event.pull_request.number }} --body "$BODY"
```

### Inline annotations

```yaml
- name: Review PR
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    claude-review pr ${{ github.event.pull_request.html_url }} \
      --format annotations \
      --output annotations.json

- name: Upload annotations
  uses: yuzutech/annotations-action@v0.4.0
  if: always()
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    title: claude-review
    input: annotations.json
```

### Security-only review for main branch merges

```yaml
name: Security Review on Merge

on:
  push:
    branches: [main]

jobs:
  security-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Install claude-review
        run: |
          curl -sSL https://github.com/critbot/claude-review/releases/latest/download/claude-review-linux-amd64.tar.gz | tar xz
          sudo mv claude-review /usr/local/bin/

      - name: Security review of merged changes
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude-review diff HEAD~1 \
            --focus security \
            --model claude-sonnet-4-6 \
            --format json \
            --output security-review.json
```

## GitLab CI

```yaml
# .gitlab-ci.yml
code_review:
  stage: review
  image: alpine:latest
  before_script:
    - apk add --no-cache curl
    - |
      curl -sSL https://github.com/critbot/claude-review/releases/latest/download/claude-review-linux-amd64.tar.gz | tar xz
      mv claude-review /usr/local/bin/
  script:
    - |
      claude-review pr $CI_MERGE_REQUEST_PROJECT_URL/-/merge_requests/$CI_MERGE_REQUEST_IID \
        --format json \
        --output review.json
    - |
      CRITICAL=$(jq '.summary.critical' review.json)
      if [ "$CRITICAL" -gt 0 ]; then
        cat review.json | jq '.findings[] | select(.severity == "critical")'
        exit 1
      fi
  artifacts:
    paths:
      - review.json
    when: always
  only:
    - merge_requests
  variables:
    ANTHROPIC_API_KEY: $ANTHROPIC_API_KEY
    GITLAB_TOKEN: $GITLAB_TOKEN
```

## Cost management in CI

To avoid unexpected API costs in CI:

### Use --estimate first (for budget-conscious pipelines)

```yaml
- name: Estimate review cost
  run: |
    ESTIMATE=$(claude-review pr $PR_URL --estimate 2>&1)
    echo "$ESTIMATE"

    # Parse estimated cost and gate on it
    COST=$(echo "$ESTIMATE" | grep -oP '\$\K[0-9.]+' | head -1)
    if (( $(echo "$COST > 5.0" | bc -l) )); then
      echo "Estimated cost $COST exceeds $5 threshold, skipping review"
      exit 0
    fi
```

### Set max_cost_usd in config

```json
{
  "max_cost_usd": 3.00
}
```

The review aborts before making API calls if the estimate exceeds this.

### Use Haiku for most PRs, Sonnet only for large ones

```yaml
- name: Determine model based on diff size
  run: |
    LINES=$(git diff origin/main...HEAD | wc -l)
    if [ "$LINES" -gt 2000 ]; then
      MODEL="claude-sonnet-4-6"
    else
      MODEL="claude-haiku-4-5-20251001"
    fi

    claude-review pr $PR_URL --model $MODEL --format json --output review.json
```

## Required secrets

| Secret | Used for |
|--------|----------|
| `ANTHROPIC_API_KEY` | All reviews (required) |
| `GITHUB_TOKEN` | GitHub PR reviews (auto-provided in GH Actions) |
| `GITLAB_TOKEN` | GitLab MR reviews |
| `BITBUCKET_TOKEN` | Bitbucket PR reviews |

In GitHub Actions, `GITHUB_TOKEN` is automatically provided by the workflow runner — no manual secret setup needed for GitHub PR reviews.

## Tips for CI

- **Cache the binary**: Download once in a setup step and cache it across runs to save time
- **Use `--agents 3`** in CI to reduce API rate limit pressure from concurrent pipelines
- **Upload review.json as an artifact** with `if: always()` so you can inspect reviews even when the job fails
- **Only run on PRs, not every push** to avoid redundant reviews of intermediate commits
