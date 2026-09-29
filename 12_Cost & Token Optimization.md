# 12_Cost & Token Optimization

# M09 — Cost & Token Optimization

> Cost baselines, rate limits by team size, 12 reduction strategies, model tier routing, and the token multiplier table.

---

## Cost Baselines

| Metric | Value |
| --- | --- |
| Average cost per developer per day | ~$6 |
| 90th percentile daily cost | <$12 |
| Average monthly cost (Sonnet 4.6) | ~$100–200/developer |
| Background token usage per session | <$0.04 |
| Agent teams multiplier | ~7× standard sessions (plan mode) |

> Costs vary by codebase size, query complexity, conversation length, number of concurrent instances, and whether used in automation.

---

## Track Your Costs

**Per-Session: `/cost`**

```
Total cost:            $0.55
Total duration (API):  6m 19.7s
Total duration (wall): 6h 33m 10.2s
Total code changes:    0 lines added, 0 lines removed
```

> `/cost` shows API token usage. Claude Max/Pro subscribers use `/stats` instead.

**Continuous monitoring** — configure the status line with `/statusline`. Set workspace spend limits via the Claude Console.

---

## Rate Limit Recommendations by Team Size

| Team Size | TPM per User | RPM per User |
| --- | --- | --- |
| 1–5 | 200k–300k | 5–7 |
| 5–20 | 100k–150k | 2.5–3.5 |
| 20–50 | 50k–75k | 1.25–1.75 |
| 50–100 | 25k–35k | 0.62–0.87 |
| 100–500 | 15k–20k | 0.37–0.47 |
| 500+ | 10k–15k | 0.25–0.35 |

> TPM per user decreases as the team grows. Limits apply at **organisation level**, not per individual.

---

## 12 Strategies to Reduce Token Usage

1. **Manage context proactively** — `/clear` between unrelated tasks, `/compact Focus on API changes`, `/context` to see usage.
2. **Choose the right model** — Haiku (simple subagent tasks, cheapest), Sonnet (most coding), Opus (complex architecture). Switch with `/model`; set subagent `model: haiku`.
3. **Reduce MCP server overhead** — MCP adds tool definitions to every request. Use `/mcp` to check costs, disconnect unused servers, prefer CLI tools, and tune tool search (`ENABLE_TOOL_SEARCH=auto:5`).
4. **Install code intelligence plugins** — one LSP call (go-to-def, find-refs) replaces grep + reading many candidate files.
5. **Offload processing to hooks** — e.g. a PreToolUse hook filters test output to failures only before it enters context.
6. **Move instructions from CLAUDE.md to skills** — keep CLAUDE.md under ~500 lines; use `disable-model-invocation: true` for zero cost until invoked.
7. **Adjust extended thinking** — lower effort via `/effort` or `/model`, disable via `/config`, cap with `MAX_THINKING_TOKENS=8000`.
8. **Delegate verbose operations to subagents** — verbose test output stays in the subagent; the main conversation gets a summary.
9. **Manage agent team costs** — use Sonnet for teammates, keep teams small, keep spawn prompts focused, clean up when done.
10. **Write specific prompts** — "add input validation to the login function in auth.ts" beats "improve this codebase".
11. **Use plan mode for complex tasks** — `Shift+Tab` → Plan Mode; explore before implementing to prevent expensive re-work.
12. **Test incrementally** — write one file → test → continue; catches issues early when cheap to fix.

---

## Model Tier Routing

| Tier | Model | Use For | Cost |
| --- | --- | --- | --- |
| Haiku | claude-haiku-4-5 | File lookups, simple transforms, grep | ~15× cheaper than Opus |
| Sonnet | claude-sonnet-4-6 | Implementation, debugging, review | Mid-tier |
| Opus | claude-opus-4-7 | Architecture decisions, synthesis, complex reasoning | Highest capability |

**Routing patterns** — set the model per subagent in frontmatter:

```yaml
---
name: file-scanner
model: claude-haiku-4-5-20251001   # Haiku for file lookups
description: Scans files for patterns and returns results
---
```

```yaml
---
name: architect-review
model: claude-opus-4-7-20260301    # Opus for synthesis only
description: Reviews architecture proposals and identifies trade-offs
---
```

> **Best practice:** Keep the main session on Sonnet. Delegate grep-and-summarise work to Haiku subagents. Reserve Opus for tasks that genuinely require deep reasoning.

---

## Context Budget: Token Multiplier Table

| Pattern | Token Multiplier vs Single Thread |
| --- | --- |
| Single-thread session | 1× |
| Plan mode (read-only exploration) | ~2× |
| Two-subagent team | ~3.5× |
| Five-subagent team (typical parallel task) | ~7× |
| Ten-subagent team | ~15× |

> **Cost trade-off:** Subagent-heavy workflows consume ~7× the tokens of a single-thread session. The parallelism benefit (wall-clock time) is real, but the trade-off must be deliberate.

### Practical Mitigations

| Action | Token Saving |
| --- | --- |
| `/clear` between unrelated tasks | 30–50% per-message reduction |
| Haiku subagents for file operations | ~15× cheaper than Opus |
| Hook-based log filtering | Eliminates verbose output from context |
| Security guidance plugin overhead | +~5% per turn |
| Security guidance plugin benefit | Prevents expensive late-stage security reviews |

---

## Auto-Optimisations

Claude Code automatically applies:

| Optimisation | How It Works |
| --- | --- |
| **Prompt caching** | Repeated content (system prompts) is cached, reducing costs |
| **Auto-compaction** | Summarises conversation when approaching context limits |
| **Tool search** | Defers MCP tools beyond 10% context, loading on-demand |