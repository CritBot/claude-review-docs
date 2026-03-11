# diff

Review a local git diff: staged changes, a commit range, or a branch comparison.

## Usage

```
claude-review diff [ref] [flags]
```

If `ref` is omitted, reviews the staged diff (`git diff --cached`).

## Flags

| Flag | Description |
|------|-------------|
| `--files <pattern>` | Restrict review to files matching a glob pattern |
| `--format <fmt>` | Output format: `markdown` (default), `json`, `annotations` |
| `--output <file>` | Output file (default: `REVIEW.md`) |
| `--model <model>` | Claude model to use |
| `--agents <n>` | Number of parallel finder agents |
| `--focus <areas>` | Comma-separated focus areas |
| `--estimate` | Print cost estimate and exit without running |
| `--memory` | Enable memory-augmented review |
| `--verbose` | Show per-agent debug output |

## Examples

### Review staged changes

```bash
git add .
claude-review diff
# shorthand: just run claude-review
```

### Review the last commit

```bash
claude-review diff HEAD~1
```

### Review the last N commits

```bash
claude-review diff HEAD~3
```

### Review a branch diff

```bash
claude-review diff main..feature/my-branch
```

This is equivalent to `git diff main..feature/my-branch`.

### Review specific files only

```bash
# Only review Go source files
claude-review diff HEAD~1 --files "*.go"

# Only review files in the auth package
claude-review diff --files "internal/auth/**"
```

### Estimate cost before running

```bash
claude-review diff main..feature --estimate
```

Output:
```
Estimated cost for this review:
  Diff size: 4,823 chars (~1,206 tokens per agent)
  Agents: 5 finders + verifier + ranker
  Model: claude-haiku-4-5-20251001
  Estimated cost: $0.18 – $0.32
```

### JSON output for CI

```bash
claude-review diff HEAD~1 --format json --output review.json
```

### GitHub Checks annotations

```bash
claude-review diff HEAD~1 --format annotations --output annotations.json
```

See [Output Formats](../guides/output-formats.md) for annotation JSON structure.

### Focused security review with Sonnet

```bash
claude-review diff main..release/v2 \
  --focus security \
  --model claude-sonnet-4-6 \
  --output security-review.md
```

### Memory-augmented review

```bash
claude-review diff HEAD~1 --memory
```

Before sending prompts to finder agents, the memory layer retrieves relevant past findings from the same files and injects them as context. New findings from this run are stored in the memory DB after the review completes.

[Learn more about memory →](../memory/how-it-works.md)

## How ref resolution works

| Input | Equivalent git command |
|-------|----------------------|
| _(none)_ | `git diff --cached` (staged) |
| `HEAD~1` | `git diff HEAD~1 HEAD` |
| `HEAD~3` | `git diff HEAD~3 HEAD` |
| `main..feature` | `git diff main..feature` |
| `a1b2c3d` | `git diff a1b2c3d HEAD` |

## Output

By default, a `REVIEW.md` file is written to the current directory. The terminal shows a summary table and the path to the report.

```
✓ Review complete

Severity   Count
🔴 Critical  1
🟠 High      3
🟡 Medium    7
💡 Suggestion 2

Cost: $0.31 · Time: 42s
Report: REVIEW.md
```
