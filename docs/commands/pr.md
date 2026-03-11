# pr

Review a pull request or merge request from GitHub, GitLab, or Bitbucket.

## Usage

```
claude-review pr <url|number> [flags]
```

## Arguments

| Argument | Description |
|----------|-------------|
| `url` | Full URL to the PR/MR/PR |
| `number` | PR/MR number — auto-detects platform from `git remote get-url origin` |

## Supported platforms

| Platform | URL format | Shorthand |
|----------|-----------|-----------|
| GitHub | `https://github.com/org/repo/pull/123` | `123` |
| GitLab | `https://gitlab.com/org/project/-/merge_requests/45` | `45` |
| GitLab (self-hosted) | `https://gitlab.company.com/org/project/-/merge_requests/45` | `45` |
| Bitbucket | `https://bitbucket.org/workspace/repo/pull-requests/7` | `7` |

## Flags

Same as [`diff`](diff.md), plus:

| Flag | Description |
|------|-------------|
| All `diff` flags | `--format`, `--output`, `--model`, `--agents`, `--focus`, `--estimate`, `--memory`, `--verbose` |

## Authentication

Set the appropriate token for your platform:

=== "GitHub"
    ```bash
    export GITHUB_TOKEN=ghp_...
    # or in config: "github_token": "ghp_..."
    ```

=== "GitLab"
    ```bash
    export GITLAB_TOKEN=glpat-...
    # or in config: "gitlab_token": "glpat-..."
    ```

=== "Bitbucket"
    ```bash
    export BITBUCKET_TOKEN=username:app_password
    # or in config: "bitbucket_token": "..."
    ```

For GitHub Actions, `GITHUB_TOKEN` is automatically available from the workflow context — no extra setup needed.

## Examples

### GitHub PR by full URL

```bash
claude-review pr https://github.com/org/repo/pull/123
```

### GitHub PR by number (inside the repo)

```bash
# Must be run from within the git repo that has the GitHub remote
claude-review pr 123
```

The tool reads `git remote get-url origin`, detects the GitHub remote, and constructs the full URL automatically.

### GitLab MR

```bash
claude-review pr https://gitlab.com/myorg/myproject/-/merge_requests/45
```

### Self-hosted GitLab

```bash
# Set your base URL in config
# "gitlab_base_url": "https://gitlab.company.com"

claude-review pr https://gitlab.company.com/org/project/-/merge_requests/12
```

### Bitbucket PR

```bash
claude-review pr https://bitbucket.org/workspace/repo/pull-requests/7
```

### PR review with JSON output

```bash
claude-review pr 123 --format json --output review.json
```

### PR review with annotations (for GitHub Checks)

```bash
claude-review pr 123 --format annotations --output annotations.json
```

See [CI Integration](../guides/ci-integration.md) for how to upload annotations as a GitHub Check.

### Cost estimate before running

```bash
claude-review pr 123 --estimate
```

### Security-focused review of a large PR

```bash
claude-review pr https://github.com/org/repo/pull/456 \
  --focus security \
  --model claude-sonnet-4-6 \
  --agents 3
```

## How the diff is fetched

For GitHub, `claude-review` calls the GitHub Compare API with pagination to fetch the full unified diff, including all files changed in the PR. It handles PRs with large numbers of files by iterating through all pages.

For GitLab and Bitbucket, the respective APIs are called similarly — paginated diff fetching ensures large MRs are reviewed completely.

The fetched diff is then parsed and passed to the same multi-agent pipeline used for local diffs.

## SSH remotes

SSH remote URLs (`git@github.com:org/repo.git`) are automatically converted to HTTPS for URL construction. No special configuration needed.
