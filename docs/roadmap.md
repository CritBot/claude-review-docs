# Roadmap

## Released

### v0.1.0 ✅

Core multi-agent pipeline: parallel finder agents (logic, security, performance, types, tests), verifier, and ranker. GitHub, GitLab, and Bitbucket PR support. Markdown, JSON, and annotations output. Cost transparency with `--estimate`. Pre-commit hook via `install-hook`. `--files` flag for targeted review. PR number shorthand (auto-detects remote). MIT licensed.

### v1.1.0 ✅

Always-on memory layer. SQLite-backed persistent findings stored at `.claude-review/memory.db` per repo. On-wake consolidation trigger — fires in the background on any `claude-review` command invocation when the time or volume threshold is met (30 minutes elapsed, or 10+ new findings). Cross-PR pattern detection via a consolidation agent that reads metadata only — no source code. DB pruning to keep the database small. `memory` subcommands (`start`, `stop`, `status`, `clear`, `install`). `insights` command for plain-English cross-PR pattern summaries.

---

## Planned

### v1.2.0

**`--fix` Auto-Apply**

Parse the `suggested_fix` field from each finding and apply it to the source file directly. Before applying, show a unified diff of the proposed change and prompt for per-finding confirmation. In CI, pass `--yes` to apply all non-interactively. The patcher verifies the target line content still matches what the agent saw before writing — if the file has changed, it aborts rather than applying a stale patch.

---

**`.reviewrc.yml` Custom Rules**

A YAML file at your repo root that injects custom review rules directly into every finder agent's prompt. Rules specify an ID, which focus area they apply to, a severity, a description of what to look for, and an optional example. Rules are merged with a global `~/.claude-review/rules.yml` (local takes precedence).

Example:

```yaml
# .reviewrc.yml
rules:
  - id: no-raw-sql
    focus: security
    severity: high
    description: "All SQL queries must use parameterized statements. Flag any string concatenation into a SQL query."
    example: "db.Query(\"SELECT * FROM users WHERE id = \" + id)"

  - id: require-request-id
    focus: logic
    severity: medium
    description: "Every HTTP handler must extract or generate a request ID and attach it to the context."

  - id: no-fmt-println-in-handlers
    focus: logic
    severity: suggestion
    description: "HTTP handlers must use structured logging, not fmt.Println."
```

Rules are injected as a `## Custom Rules` block alongside the memory context before each finder agent runs. The `rules validate` command checks your file for schema errors. The `rules init` command scaffolds a starter file for common stacks.

Planned CLI additions:
```bash
claude-review rules validate       # schema-check .reviewrc.yml
claude-review rules list           # show active rules (global + local merged)
claude-review rules init           # scaffold a starter .reviewrc.yml
```

---

**Context Scraper**

Eliminates the cold-start problem for the memory layer. On a fresh install, memory is empty and the first few reviews get no context. The Context Scraper bootstraps it by reading your repo's existing PR history from GitHub, GitLab, or Bitbucket.

What it scrapes:
- Closed PR titles, descriptions, and outcomes (merged / rejected)
- Inline review comments and their file/line locations
- Review decisions (approved, changes requested) and who made them

A Context Ingest Agent processes the scraped data and converts it into memory-format signals: which files get the most review comments, which patterns reviewers flag repeatedly, which suggestions were consistently accepted or rejected. These are stored in a `context_events` table in `memory.db` and feed into the consolidation agent alongside regular findings.

After one scrape, the memory layer understands your team's review culture from day one — before you've run a single `claude-review diff`.

```bash
claude-review context scrape            # scrape last 100 closed PRs
claude-review context scrape --limit 500 --since 2025-01-01
claude-review context scrape --dry-run  # show what would be ingested, don't write
```

---

### v1.3.0

**CVE Database Integration**

Elevates security findings from "this looks suspicious" to "this matches a known vulnerability class". After the verifier phase, each `security` finding is passed to a CVE Lookup Agent that searches OSV (Google's open-source vulnerability database) and NVD.

The framing is deliberate: results say "this pattern is **similar to** CVE-2021-44228" — not "this IS CVE-2021-44228". The agent includes a confidence note and links to the CVE entry. This is especially useful for dependency-related findings.

CVE lookups are cached in `memory.db` with a 7-day TTL to avoid redundant API calls. Use `--no-cve` to disable in air-gapped environments.

In Markdown output, CVE references appear as a collapsible block below the finding:

```markdown
### SQL injection via unsanitized user input

**File**: `api/handlers/user.go` · **Line**: 43 · **Confidence**: 92%

...finding description...

??? note "Related vulnerability classes"
    - [CWE-89: SQL Injection](https://cwe.mitre.org/data/definitions/89.html)
    - Similar pattern to [CVE-2023-XXXXX](https://osv.dev/vulnerability/...) — SQL injection in Go HTTP handlers via unparameterized queries
```

---

**Plugin / Recipe System**

Named bundles of finder prompts, custom rules, and CVE category filters — shareable, composable, and community-driven.

A recipe is a single YAML file:

```yaml
# ~/.claude-review/recipes/fintech-pci.yml
name: fintech-pci
version: 1.0.0
description: "PCI-DSS focused review for fintech applications"
focus:
  - security
  - logic
  - types
rules:
  - id: no-card-data-in-logs
    focus: security
    severity: critical
    description: "PAN, CVV, and expiry data must never appear in log output."
  - id: require-audit-log
    focus: logic
    severity: high
    description: "All financial state mutations must write an audit log entry."
cve_categories:
  - CWE-312   # Cleartext storage of sensitive information
  - CWE-359   # Exposure of private information
prompt_extras:
  - "This codebase handles payment card data. Apply PCI-DSS DSS v4.0 requirements throughout."
```

Usage:
```bash
claude-review diff --recipe fintech-pci
claude-review diff --recipe hipaa
claude-review diff --recipe react-performance
claude-review diff --recipe ./my-custom-recipe.yml
```

Built-in recipes: `default`, `fintech-pci`, `hipaa`, `react-performance`, `go-concurrency`.

Community recipes live in `critbot/recipes` — each recipe is a standalone `.yml` file. A future `recipe install` command will fetch them directly.

```bash
claude-review recipe list               # list available recipes
claude-review recipe show fintech-pci   # print full YAML
claude-review recipe init my-recipe     # scaffold a new recipe file
```

---

**Azure DevOps Support**

Review Azure DevOps pull requests directly:

```bash
claude-review pr https://dev.azure.com/org/project/_git/repo/pullrequest/42
```

Requires `AZURE_DEVOPS_TOKEN` (Personal Access Token with Code read permission) or `azure_devops_token` in config.

---

### v2.0.0

**Self-Hosting / On-Prem**

Enterprise deployment with alternative AI backends and multi-user support.

*Alternative AI backends*: An `LLMClient` interface abstracts the Anthropic HTTP client. Drop-in implementations for AWS Bedrock (Claude model ARNs) and GCP Vertex AI let you route API calls through your existing cloud infrastructure. Configure with `llm_backend: bedrock` or `llm_backend: vertex` and the corresponding credential fields.

*Pluggable storage*: A `Store` interface behind `memory.DB` allows swapping SQLite for Postgres for team-shared memory. One team's review history, accessible to every engineer.

*Server mode*: `claude-review server` exposes an HTTP API that accepts diff payloads and returns JSON findings. Enables web UIs, IDE plugins, and custom integrations without spawning a CLI subprocess.

*Multi-user*: Per-user API key isolation. Optional org-level shared memory. RBAC for sensitive commands (`insights`, `memory clear`).

*Web UI*: Minimal dashboard for `insights` output, finding history, and false-positive management — for teams that prefer a browser over a terminal.

All v1.x code compiles and runs without the server component. Self-hosting is additive, not a fork.

---

**Stable Public Go API & Plugin Interface**

A stable `pkg/` API for embedding `claude-review` in other Go tools. Exported types, a `Review(ctx, payload, cfg)` entry point, and a plugin interface for custom focus-area agents. Full semantic versioning commitment, deprecation policy, and migration guide from v0.x.
