# Quick Start — Claude Code Masterclass

> **Core Insight:** Claude Code is an agentic workflow layer, not a chat box. Context is the most important resource to manage — it's why subagents, agent teams, and memory exist. Master context management and everything else follows.

> **Teaching Principle:** Claude Code is most useful when teams stop treating it as a single prompt box and start treating it as an engineering workflow layer. Put durable knowledge in CLAUDE.md, repeatable workflows in skills, use subagents for parallel work, hooks for guardrails, and package shared practices as plugins.

---

## Table of Contents — Learning Path

A 7-day recommended ramp from first session to production patterns.

- [Days 1–2 · Foundation](#days-12--foundation)
- [Days 3–4 · Orchestration](#days-34--orchestration)
- [Days 5–6 · Automation &amp; Guardrails](#days-56--automation--guardrails)
- [Day 7+ · Production &amp; Integration](#day-7--production--integration)
- [Reference](#reference)

---

## Days 1–2 · Foundation

Get comfortable with how Claude Code works and how it remembers.

- **Module 1 — Agentic Loop & Tools** — Three-phase loop, 25+ tools across 7 categories, permission model, context cost.
- **Module 2 — Extension Features Comparison** — Decision framework for Skills vs Subagents vs CLAUDE.md vs MCP vs Hooks.
- **Module 3 — Memory & CLAUDE.md** — 4-layer memory system, CLAUDE.md at 3 scopes, auto-memory, 500-line discipline.
- **Lab 1 — Agentic Loop & Context** — Debug intermittent 500 errors using Explore→Plan→Code without burning context.
- **Lab 2 — CLAUDE.md & Memory** — Configure CLAUDE.md and auto-memory for a Python monorepo with 3 services.

## Days 3–4 · Orchestration

Coordinate multiple agents to work in parallel.

- **Module 4 — Subagents** — Built-in subagents, YAML frontmatter, scoped MCP, hooks, 8 ready-to-use templates.
- **Module 5 — Agent Teams** *(Experimental)* — Team Lead + Teammates, task management, mailbox communication, quality gates.
- **Module 6 — Design Patterns & Best Practices** — The 10 official best practices, 7 multi-agent patterns, context management strategies, common failure modes.
- **Lab 3 — Subagents** — Dispatch 3 parallel review subagents for a PR touching auth, payments, and UI.

## Days 5–6 · Automation & Guardrails

Automate repeatable work and add safety rails.

- **Module 7 — Hooks** — 22 hook events, 4 hook types, exit code semantics, 15 ready-to-use patterns.
- **Module 8 — Slash Commands & Skills** — 15 built-in commands, SKILL.md format, legacy commands, SDK integration.
- **Module 9 — Cost & Token Optimization** — 12 reduction strategies, model tier routing, rate limits, token multiplier table.
- **Lab 4 — Hooks** — Write PreToolUse hook blocking .env edits and Stop hook enforcing test runs.
- **Lab 5 — Cost & Token Optimization** — Diagnose 3× budget overrun in a 20-engineer team.

## Day 7+ · Production & Integration

Ship, observe, and integrate Claude Code into your stack.

- **Module 10 — Observability** — OpenTelemetry setup, 8 metrics, 4 event types, dashboards, cost alerts.
- **Module 11 — Headless Mode & CI/CD** — Non-interactive `claude -p`, output formats, GitHub Actions, GitLab CI.
- **Module 12 — Sandboxing & Security** — Filesystem & network isolation, OS-level enforcement, prompt injection protection.
- **Module 13 — MCP Integration** — 4 transport types, scope hierarchy, tool search, elicitation, resources.
- **Module 14 — Plugins & Marketplace** — 170+ plugins, plugin.yaml schema, Security Guidance deep-dive, building your own.
- **Lab 6 — Sandboxing & Security** — Enable sandbox mode, verify boundaries, simulate prompt injection.
- **Lab 7 — Headless & CI/CD** — Wire Claude Code into GitHub Actions PR review workflow.
- **Lab 8 — MCP & Plugins** — Connect 3 MCP servers, measure overhead, scope to subagents.

---

## Reference

Quick lookup for the whole curriculum.

- **Feature Matrix** — All 14 concepts mapped to modules, labs, assets, and source URLs.
- **Learning Path** — 7-day recommended ramp from first session to production patterns.

**At a glance:** 14 Modules · 8 Labs across 4 phases.