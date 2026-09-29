# 27_E02_Dynamic Workflows

# E02 — Dynamic Workflows in Claude Code

> How Claude Code writes its own harness on the fly — custom-built for every task. Covers the six core patterns, real-world use cases, and how to build and share workflows. (Published June 2, 2026 — Thariq Shihipar & Sid Bidasaria)

---

## What Are Dynamic Workflows?

The default harness is excellent for everyday coding. But for specialized domains — deep research, large refactors, security analysis, evaluation pipelines — a *custom* harness consistently outperforms the default.

Dynamic workflows let Claude Code write that custom harness itself, on the fly. You describe the task; Claude generates a JavaScript workflow file that spawns and coordinates multiple subagents, each with isolated context, to accomplish complex multi-step objectives.

**Example prompts that trigger dynamic workflow generation:**

- "This test fails maybe 1 in 50 runs. Set up a workflow to reproduce it. Form competing theories about the race condition…"
- "Mine my last 30 sessions for patterns where you kept making the same mistake. Add them to CLAUDE.md."
- "Review this business plan from three perspectives: investor skeptic, target customer, and competitor."
- "Rank these 200 candidates against the job description. Interview me using the AskUserQuestion tool for edge cases."
- "Refactor all uses of the User model to use UserV2. Verify each change with adversarial review before merging."

---

## Why Subagents Solve Problems Single Agents Can't

Three failure modes plague single-agent tasks on complex work; dynamic workflows address all three by design:

- **Agentic Laziness** — a single agent stops before completing complex multi-part tasks. Separate subagents with fresh context windows stay on-task until their objective is complete.
- **Self-Preferential Bias** — an agent evaluating its own output favors its own results. A separate verifier subagent with no knowledge of the original work produces unbiased evaluation.
- **Goal Drift** — over long sessions, agents gradually lose fidelity to objectives. Short-lived subagents with explicit, scoped goals don't accumulate drift.

---

## The Six Core Workflow Patterns

Every dynamic workflow is some combination of these six primitives.

| Pattern | What It Does | Best For |
| --- | --- | --- |
| **1. Classify-and-Act** | A fast, cheap agent classifies input; deeper agents handle each class | Bug triage, support ticket routing, PR categorization |
| **2. Fan-Out-and-Synthesize** | N agents work in parallel; one synthesis agent consolidates | Deep research, codebase-wide refactors, multi-source analysis |
| **3. Adversarial Verification** | A "skeptic" agent tries to disprove the "implementer" agent's work; majority vote decides | Security review, factual verification, patch validation |
| **4. Generate-and-Filter** | Generation agent produces 5–10 candidates; evaluation agent scores against a rubric | Naming, design exploration, message generation |
| **5. Tournament** | Competing agents judged pairwise (bracket-style); more reliable than absolute scoring at scale | Candidate ranking, dataset sorting, option evaluation (1000+ rows) |
| **6. Loop-Until-Done** | Continue spawning agents until stop conditions are met; combine with `/goal` and `/loop` | Continuous monitoring, iterative improvement, long-running research |

---

## High-Value Enterprise Use Cases

**Large-Scale Migrations & Refactors** — Bun's Zig-to-Rust rewrite is the canonical example. Strategy: break tasks into atomic steps → spawn subagents per fix in isolated worktrees → adversarially review each change → merge only verified work.

```
1. Classifier agent: scan codebase, list all ZigModule usages (output: task list)
2. Fan-out: spawn one subagent per module in its own worktree
3. Each subagent: rewrite module, write tests, run CI
4. Adversarial: fresh agent reviews diff in each worktree
5. Merge queue: only diffs that pass adversarial review get PR'd
```

**Root-Cause Investigation** — self-preferential bias is fatal for debugging. Generate independent hypotheses from disjoint evidence:

- Agent A reads only logs
- Agent B reads only recent commits
- Agent C reads only test failures
- Synthesis agent reconciles hypotheses and proposes the most likely root cause

**Memory & Rule Adherence** — create one verification agent per rule in CLAUDE.md; each checks only its assigned rule. Reverse approach: mine recent sessions, cluster corrections with parallel agents, verify with skeptic personas, then propose CLAUDE.md additions — a self-improving harness.

**Model Routing for Cost Optimization** — a classifier agent determines task complexity, then routes: simple → Haiku, moderate → Sonnet, complex → Opus. Can reduce token costs by 50–70% on mixed workloads.

---

## Saving and Sharing Workflows

```
# During a session: press "s" to save current workflow
# Saved to: ~/.claude/workflows/

# To share org-wide: place .js file in your plugin's skills folder
my-org-plugin/
├── plugin.yaml
└── skills/
    ├── deep-research.md        # skill that invokes the workflow
    └── deep-research.js        # the workflow JavaScript file

# Reference in SKILL.md to make it invocable as /deep-research
```

> **Token Budget Discipline:** Workflows are token-intensive by design. Set explicit budgets ("use no more than 10k tokens per subagent"). Combine with `/loop` for scheduled recurring execution.

> **When NOT to use workflows:** Regular coding tasks don't need a panel of 5 reviewers. Workflows are for high-value, complex tasks where the extra token cost is justified by quality improvement. Start simple; add complexity only when single-agent approaches repeatedly fall short.

> Source: *A harness for every task: dynamic workflows in Claude Code* — Anthropic Blog, June 2, 2026.