# install-hook

Install or remove `claude-review` as a git pre-commit hook.

## Usage

```
claude-review install-hook [--remove]
```

## Flags

| Flag | Description |
|------|-------------|
| `--remove` | Remove the pre-commit hook |

## Description

The pre-commit hook runs `claude-review` automatically every time you run `git commit`. If critical findings are detected, the commit is blocked until you address them (or explicitly bypass with `git commit --no-verify`).

## Install

```bash
cd your-project
claude-review install-hook
```

This writes a shell script to `.git/hooks/pre-commit` in the current repository.

## What the hook does

When `git commit` is run, the hook:

1. Checks that `ANTHROPIC_API_KEY` is set
2. Runs `claude-review diff` on the staged changes
3. If any **critical** or **high** severity findings are found, prints the report and exits with code 1 (blocking the commit)
4. If only medium/suggestion findings exist, shows a summary but allows the commit to proceed

## Remove

```bash
claude-review install-hook --remove
```

## Bypass for a single commit

If you need to commit despite failing review (e.g., a work-in-progress commit):

```bash
git commit --no-verify -m "wip: work in progress"
```

## Customizing behavior

The hook respects your `claude-review.config.json`. To reduce noise in pre-commit runs, consider a project config that uses fewer agents and focuses on critical issues only:

```json
{
  "agents": 2,
  "focus": ["security", "logic"],
  "confidence_threshold": 0.90,
  "model": "claude-haiku-4-5-20251001"
}
```

This keeps hook runs fast and cheap while still catching the most important issues before they land.

## Team setup

To ensure everyone on your team has the hook installed, add setup instructions to your `CONTRIBUTING.md` or `Makefile`:

```makefile
# Makefile
setup:
    claude-review install-hook
    @echo "Pre-commit hook installed"
```

!!! note
    Git hooks are not tracked by version control. Each developer must run `install-hook` themselves. If you want hooks that are enforced at the repo level, consider CI instead — see [CI Integration](../guides/ci-integration.md).
