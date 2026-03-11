# CI Integration

`claude-review` is designed to fit naturally into CI pipelines. This page covers integration patterns for GitHub Actions and GitLab CI, including how to carry the memory layer across pipeline runs.

---

## Memory in CI: two modes

Before setting up CI, decide how you want memory to work:

### Local-only (default)

`.claude-review/` is gitignored. The memory DB lives only on developer machines. CI runs are stateless — every run starts cold with no prior findings.

**Good for**: teams where only developers run reviews locally, CI is just for enforcement.

### Shared / CI mode

`memory.db` is committed to the repo (or stored on a separate git branch). CI picks it up automatically on checkout, runs the review with full memory context, and writes accumulated findings back.

**Good for**: teams where CI is the primary review path, or where you want every developer and CI run to share the same growing knowledge of the codebase.

The rest of this page focuses on shared mode — it's the setup that makes memory actually useful in CI.

---

## Shared mode: the orphan branch approach

Committing `memory.db` directly to your main branch would bloat git history with binary file churn on every PR. The clean solution is a separate **orphan branch** (`claude-review-memory`) that stores only the DB file. It has no shared history with `main`, so it never inflates your main branch's object storage.

### One-time setup

```bash
# Create the orphan branch locally
git checkout --orphan claude-review-memory
git rm -rf .
echo "claude-review memory storage" > README.md
git add README.md
git commit -m "Initialize claude-review memory branch"
git push origin claude-review-memory

# Return to your working branch
git checkout main
```

That's it. The `main` branch (and your `.gitignore`) don't change.

---

## GitHub Actions

### Full PR review with memory

```yaml
# .github/workflows/review.yml
name: Code Review

on: [pull_request]

permissions:
  contents: write   # needed to push memory.db back to orphan branch
  pull-requests: read

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # full history needed for branch operations

      - name: Restore memory DB
        run: |
          git fetch origin claude-review-memory 2>/dev/null || true
          if git cat-file -e origin/claude-review-memory:memory.db 2>/dev/null; then
            mkdir -p .claude-review
            git show origin/claude-review-memory:memory.db > .claude-review/memory.db
            echo "Memory DB restored ($(wc -c < .claude-review/memory.db) bytes)"
          else
            echo "No memory DB yet — first run will be cold"
          fi

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
            --memory \
            --format json \
            --output review.json

      - name: Upload review artifact
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: code-review
          path: review.json

      - name: Persist memory DB
        # Only push memory back on main branch merges to avoid race conditions
        # from concurrent PRs overwriting each other's DB updates.
        if: github.ref == 'refs/heads/main' && always()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          if [ ! -f .claude-review/memory.db ]; then
            echo "No memory DB to persist"
            exit 0
          fi

          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          # Switch to orphan branch, update DB, push
          git fetch origin claude-review-memory 2>/dev/null || true
          git checkout claude-review-memory 2>/dev/null || \
            git checkout --orphan claude-review-memory

          cp .claude-review/memory.db memory.db
          git add memory.db
          git commit -m "Update claude-review memory [skip ci]" || echo "No changes"
          git push origin claude-review-memory

          git checkout ${{ github.sha }} -- .
```

### Block PRs on critical findings

```yaml
      - name: Enforce quality gate
        run: |
          CRITICAL=$(jq '.summary.critical' review.json)
          if [ "$CRITICAL" -gt 0 ]; then
            echo "❌ $CRITICAL critical finding(s) — review REVIEW.md"
            exit 1
          fi
```

### Post findings as a PR comment

```yaml
      - name: Comment on PR
        if: always()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          CRITICAL=$(jq '.summary.critical' review.json)
          HIGH=$(jq '.summary.high' review.json)
          MEDIUM=$(jq '.summary.medium' review.json)
          COST=$(jq -r '.cost.estimated_usd' review.json | awk '{printf "%.3f", $1}')

          FINDINGS=$(jq -r '
            .findings[]
            | select(.severity == "critical" or .severity == "high")
            | "**\(.severity | ascii_upcase)** `\(.file):\(.line)` — \(.title)"
          ' review.json | head -10)

          BODY="## 🤖 claude-review

          | 🔴 Critical | 🟠 High | 🟡 Medium |
          |------------|---------|----------|
          | $CRITICAL | $HIGH | $MEDIUM |

          $FINDINGS

          *\$$COST · ${{ github.sha }} · [full report]()*"

          gh pr comment ${{ github.event.pull_request.number }} --body "$BODY"
```

---

## GitLab CI

```yaml
# .gitlab-ci.yml
code_review:
  stage: review
  image: alpine:latest
  before_script:
    - apk add --no-cache curl git jq
    - |
      curl -sSL https://github.com/critbot/claude-review/releases/latest/download/claude-review-linux-amd64.tar.gz | tar xz
      mv claude-review /usr/local/bin/
  script:
    # Restore memory DB from orphan branch
    - |
      git fetch origin claude-review-memory 2>/dev/null || true
      if git cat-file -e origin/claude-review-memory:memory.db 2>/dev/null; then
        mkdir -p .claude-review
        git show origin/claude-review-memory:memory.db > .claude-review/memory.db
      fi
    # Run the review
    - |
      claude-review pr "$CI_MERGE_REQUEST_PROJECT_URL/-/merge_requests/$CI_MERGE_REQUEST_IID" \
        --memory \
        --format json \
        --output review.json
    # Quality gate
    - |
      CRITICAL=$(jq '.summary.critical' review.json)
      [ "$CRITICAL" -eq 0 ] || (echo "Critical findings detected"; exit 1)
    # Persist memory DB (main branch only)
    - |
      if [ "$CI_COMMIT_BRANCH" = "main" ]; then
        git config user.name "gitlab-ci-bot"
        git config user.email "gitlab-ci-bot@noreply"
        git fetch origin claude-review-memory 2>/dev/null || true
        git checkout claude-review-memory 2>/dev/null || git checkout --orphan claude-review-memory
        cp .claude-review/memory.db memory.db
        git add memory.db
        git commit -m "Update claude-review memory [skip ci]" || true
        git push origin claude-review-memory
        git checkout "$CI_COMMIT_SHA"
      fi
  artifacts:
    paths:
      - review.json
    when: always
  only:
    - merge_requests
    - main
  variables:
    ANTHROPIC_API_KEY: $ANTHROPIC_API_KEY
    GITLAB_TOKEN: $GITLAB_TOKEN
```

---

## Why only persist on main?

PRs run concurrently. If every PR pushes its updated `memory.db` back to the orphan branch, two PRs running at the same time will race and one will overwrite the other's update.

The safe pattern: **PRs read memory (for context), but only merged commits write it back.**

This means:
- PRs get full memory context on review ✓
- Memory is updated once per merge, not per PR ✓
- No race conditions ✓

A short delay between a merge and the next PR seeing the updated memory is acceptable.

---

## Memory DB size in CI

`memory.db` is a SQLite file. After consolidation (which runs automatically on every `claude-review` invocation if the trigger conditions are met), the DB is pruned:

- Findings older than **90 days** are deleted
- Each file is capped at its **50 most recent findings**
- Consolidated insights are never pruned — they're compact text summaries

On an active repo with 10 PRs/week, expect the DB to stabilize under **1 MB** permanently. It's safe to commit this to git — the orphan branch approach ensures it never touches your main branch's history.

To check the current DB size:

```bash
ls -lh .claude-review/memory.db
sqlite3 .claude-review/memory.db "SELECT COUNT(*) FROM findings;"
```

---

## Required secrets

| Secret | Used for |
|--------|----------|
| `ANTHROPIC_API_KEY` | All reviews (required) |
| `GITHUB_TOKEN` | GitHub PR reviews + pushing memory back (auto-provided in GH Actions) |
| `GITLAB_TOKEN` | GitLab MR reviews |
| `BITBUCKET_TOKEN` | Bitbucket PR reviews |

---

## Non-memory CI setup

If you don't want shared memory in CI, the minimal setup is:

```yaml
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

- name: Upload review
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: code-review
    path: review.json
```

No memory, no orphan branch, no extra permissions. Every run is cold but still catches issues in the diff.
