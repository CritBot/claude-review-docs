# Multi-Agent Pipeline

`claude-review` runs a purpose-built multi-agent pipeline designed to maximize finding coverage while minimizing false positives and cost.

## Why multiple agents?

A single large prompt asking Claude to "find all bugs" in a diff produces mediocre results. When a model must simultaneously reason about logic bugs, security, performance, type safety, and test coverage, attention is split and important findings are missed.

`claude-review` instead assigns each concern to a **specialist agent** with a focused prompt. Each agent sees the same diff but is instructed to look for only one class of problem. This dramatically improves recall in each area.

The tradeoff is that multiple agents produce overlapping and potentially contradicting results — which is why there's a **Verifier agent** (deduplication) and a **Ranker agent** (prioritization).

## Pipeline phases

### Phase 1–3: Finder agents (parallel)

```
git diff  ──►  [Logic finder]      ──►  findings[]
          ──►  [Security finder]   ──►  findings[]
          ──►  [Performance finder]──►  findings[]
          ──►  [Types finder]      ──►  findings[]
          ──►  [Tests finder]      ──►  findings[]
```

All finder agents run **concurrently** using goroutines, bounded by a configurable semaphore (`--agents` flag). Each agent:

1. Loads its specialist prompt template (embedded in the binary)
2. Injects the diff text (truncated to model context limits if needed)
3. Optionally injects memory context (if `--memory` is enabled)
4. Calls the Anthropic API with up to 3 retries on transient errors
5. Parses the JSON response — robustly handles markdown-fenced JSON, preamble text, and malformed output

Each agent returns a list of `Finding` objects:

```json
{
  "file": "src/auth/token.go",
  "line": 87,
  "severity": "critical",
  "confidence": 0.95,
  "title": "JWT expiry is never validated",
  "description": "The decoded token's expiry claim...",
  "suggested_fix": "if claims.ExpiresAt.Unix() < time.Now().Unix() { ... }"
}
```

### Phase 4a: Verifier agent (sequential)

```
[all finder results]  ──►  [Verifier]  ──►  deduplicated findings[]
```

The Verifier agent receives all findings from all finders and:

1. **Deduplicates**: merges findings that refer to the same issue (same file + line + similar description)
2. **Filters**: removes findings below the confidence threshold (default 0.80)
3. **Validates**: flags findings that require broader context than a diff provides (marked `needs_context: true`)

If the Verifier's API call fails or returns malformed output, the pipeline falls back to a deterministic deduplication algorithm — the review always completes.

### Phase 4b: Ranker agent (sequential)

```
[verified findings]  ──►  [Ranker]  ──►  prioritized findings[]
```

The Ranker agent sorts findings by:

1. Severity (critical → high → medium → suggestion)
2. Confidence score within each severity tier
3. Critical file elevation (files named `auth`, `security`, `crypto`, `payment` are prioritized within their tier)

If the Ranker fails, findings are sorted deterministically by severity score — the output is always ordered.

## Agent prompts

Each finder has a dedicated prompt template embedded in the binary:

| Agent | Template | Focus |
|-------|----------|-------|
| Logic | `finder-logic.md` | Off-by-one errors, wrong conditionals, data flow bugs, incorrect algorithm implementations |
| Security | `finder-security.md` | OWASP Top 10, injection flaws, auth/authz bypasses, secrets in code, insecure crypto |
| Performance | `finder-performance.md` | N+1 queries, unnecessary allocations, blocking calls, inefficient data structures |
| Types | `finder-types.md` | Nil/null dereferences, type mismatches, unchecked assertions, missing error handling |
| Tests | `finder-tests.md` | Missing test coverage for new code, tests without failure paths, brittle assertions |
| Verifier | `verifier.md` | Dedup, confidence filtering, context requirements |
| Ranker | `ranker.md` | Severity sorting, critical file prioritization |

## Concurrency model

```go
// Simplified pseudocode
sem := make(chan struct{}, cfg.Agents)  // semaphore

for _, agent := range finders {
    wg.Add(1)
    go func(a Agent) {
        sem <- struct{}{}         // acquire slot
        defer func() { <-sem }() // release slot
        defer wg.Done()
        results[a.Name] = a.Run(diff, memCtx)
    }(agent)
}
wg.Wait()
```

The semaphore limits concurrent API calls to `cfg.Agents` (default 4). This prevents rate limiting while maximizing parallelism.

## Cost model

Cost is calculated from actual token usage reported by the API:

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|----------------------|
| `claude-haiku-4-5-20251001` | $0.80 | $4.00 |
| `claude-sonnet-4-6` | $3.00 | $15.00 |
| `claude-opus-4-6` | $15.00 | $75.00 |

For a typical 500-line diff reviewed with 5 Haiku agents:

- Input: ~1,500 tokens × 7 agents = ~10,500 input tokens = ~$0.008
- Output: ~500 tokens × 7 agents = ~3,500 output tokens = ~$0.014
- **Total: ~$0.02–0.05** for a typical PR

Larger PRs with Sonnet cost $1–3. The `--estimate` flag gives you a prediction before spending anything.

## Truncation

If a diff exceeds the model's context window, `claude-review` truncates it by:

1. Keeping all file headers (metadata about what changed)
2. Keeping the full content of the most-changed files
3. Truncating the least-changed files' content last

This ensures the most important changes are always fully reviewed.

## Error handling and fallbacks

| Failure | Fallback |
|---------|----------|
| Finder agent API call fails after 3 retries | Finding set for that agent is empty (others still contribute) |
| Verifier agent fails | Deterministic dedup by file+line hash |
| Ranker agent fails | Deterministic sort by severity score |
| Malformed JSON from agent | Regex extraction of JSON array from response text |

The pipeline is designed to always produce a review, even under adverse conditions.
