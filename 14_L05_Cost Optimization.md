# 14_L05_Cost Optimization

# Lab 5 — Cost & Token Optimization

> Diagnose a 3× budget overrun and reduce token spend.

---

## Scenario

Your team's Claude Code bill tripled after adding three MCP servers and a verbose CLAUDE.md. You've been asked to bring cost back under target without degrading output quality. You will measure, identify waste, and apply fixes systematically.

> **Goal:** Diagnose a 3× budget overrun in a team of 20 engineers and reduce token spend using targeted strategies in a single session.

### Setup Assumptions

- You have access to usage metrics (or will simulate via `/context` readings).
- CLAUDE.md exists and is longer than 300 lines.
- Three MCP servers are active: filesystem, database, and an internal API.
- Engineers are using Opus for all tasks including quick lookups.

---

## Exercise Steps

### Step 1 — Measure Baseline Context Overhead

```text
Run /context and report: total tokens in current context, how many are
from CLAUDE.md, how many from MCP server tool registrations, and how
many from conversation history. Give counts, not percentages.
```

### Step 2 — Identify MCP Overhead Sources

```text
List each active MCP server and how many tools it registers. Identify
which server contributes the most tokens to the context window per
session start. Recommend which to scope to subagents only.
```

### Step 3 — Trim CLAUDE.md

```text
Review CLAUDE.md and cut it to under 100 lines. Keep: project summary,
run commands, safety rules. Remove: verbose rationale, examples that
repeat what the code already shows, historical decisions that belong in
ADRs. Show a diff of removed sections.
```

### Step 4 — Set Up Model Tier Routing

```text
Write a CLAUDE.md section that tells the team which model to use for
each task type: Haiku for file lookups and grep-style searches, Sonnet
for implementation and debugging, Opus only for architecture decisions
and ambiguous cross-service changes. Include the /model command syntax.
```

### Step 5 — Measure Savings

```text
Run /clear, start a fresh session, run /context again, and report the
new token counts. Calculate percentage reduction from the baseline.
```

---

## Expected Claude Code Behaviour

Claude identifies MCP tool registration as a significant overhead source, produces a tight CLAUDE.md diff, and surfaces clear model routing guidance. The post-clear context reading shows a measurable reduction from baseline.

### Success Criteria

- Baseline `/context` reading captures token counts by source.
- The highest-overhead MCP server is identified and scoped to subagents.
- CLAUDE.md shrinks by at least 50% with no loss of mandatory conventions.
- Model routing rules are written into CLAUDE.md and are unambiguous.
- Post-clear `/context` shows lower token count and you can state the saving percentage.

> **Reflection:** `/clear` saves 30–50% of context by discarding conversation history, but it also discards working context. What is the right heuristic for when to `/clear` during a long implementation session versus staying in-context?