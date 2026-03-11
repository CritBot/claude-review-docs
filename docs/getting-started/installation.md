# Installation

## Requirements

- An [Anthropic API key](https://console.anthropic.com) — you need a paid API account to use Claude models
- For platform PR reviews: a GitHub/GitLab/Bitbucket token with read access

## macOS (Homebrew) — recommended

```bash
brew install critbot/tap/claude-review
```

This installs a pre-built binary and keeps it up to date with `brew upgrade`.

## Linux / macOS (direct download)

Download the latest binary for your platform:

=== "Linux amd64"
    ```bash
    curl -sSL https://github.com/critbot/claude-review/releases/latest/download/claude-review-linux-amd64.tar.gz | tar xz
    sudo mv claude-review /usr/local/bin/
    ```

=== "Linux arm64"
    ```bash
    curl -sSL https://github.com/critbot/claude-review/releases/latest/download/claude-review-linux-arm64.tar.gz | tar xz
    sudo mv claude-review /usr/local/bin/
    ```

=== "macOS amd64 (Intel)"
    ```bash
    curl -sSL https://github.com/critbot/claude-review/releases/latest/download/claude-review-darwin-amd64.tar.gz | tar xz
    sudo mv claude-review /usr/local/bin/
    ```

=== "macOS arm64 (Apple Silicon)"
    ```bash
    curl -sSL https://github.com/critbot/claude-review/releases/latest/download/claude-review-darwin-arm64.tar.gz | tar xz
    sudo mv claude-review /usr/local/bin/
    ```

=== "Windows amd64"

    **Option 1 — Direct download (PowerShell)**
    ```powershell
    # Download and extract
    Invoke-WebRequest -Uri "https://github.com/critbot/claude-review/releases/latest/download/claude-review-windows-amd64.zip" -OutFile claude-review.zip
    Expand-Archive claude-review.zip -DestinationPath .

    # Move to a directory on your PATH, e.g.:
    Move-Item claude-review.exe "$env:USERPROFILE\bin\claude-review.exe"
    ```

    **Option 2 — Build from source** (requires Go 1.22+, see below)

    **Setting ANTHROPIC_API_KEY on Windows:**
    ```powershell
    # Current session only
    $env:ANTHROPIC_API_KEY = "sk-ant-..."

    # Persist across sessions (user-level)
    [System.Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-ant-...", "User")
    ```

## Go install

If you have Go 1.22+ installed, the simplest cross-platform option:

```bash
go install github.com/critbot/claude-review/cmd/claude-review@latest
```

This builds from source and installs to `$GOPATH/bin` (typically `~/go/bin`).

## Build from source

Requires Go 1.22+.

```bash
git clone https://github.com/critbot/claude-review
cd claude-review
make install   # builds and installs to $GOPATH/bin
```

Or just build locally:

```bash
make build     # outputs to dist/claude-review
```

## Verify the installation

```bash
claude-review --version
```

## Set your API key

`claude-review` requires an Anthropic API key. Set it as an environment variable:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

Add this to your `~/.bashrc`, `~/.zshrc`, or equivalent shell config to persist it across sessions.

For platform-specific PR reviews, also set the appropriate token:

```bash
export GITHUB_TOKEN=ghp_...        # GitHub PRs
export GITLAB_TOKEN=glpat-...      # GitLab MRs
export BITBUCKET_TOKEN=...         # Bitbucket PRs (app password)
```

!!! tip "Per-project config"
    You can also set these in a `claude-review.config.json` file in your project root. See [Configuration](configuration.md) for details.

## Checking your setup

Run a quick test with a local diff:

```bash
# Stage any change in a git repo
echo "// test" >> README.md
git add README.md

# Run a review
claude-review

# Clean up
git restore README.md
```

If everything is working, you'll see a review report written to `REVIEW.md`.
