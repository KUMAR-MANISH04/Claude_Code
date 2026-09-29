# 02_Extension Features Comparison

# Extension Features Comparison

> A decision framework for Claude Code's six extension features — **CLAUDE.md, Skills, Subagents, Agent Teams, MCP, and Hooks**. Know when to use what.

---

## The Decision Framework

The extension layer has six features that solve different problems. Choosing the wrong one wastes tokens and adds complexity.

| I Need To… | Use |
| --- | --- |
| Set "always do X" rules for every session | **CLAUDE.md** |
| Provide reference docs Claude uses sometimes | **Skill** |
| Create a repeatable workflow invoked with `/name` | **Skill** |
| Run code in isolation without polluting context | **Subagent** |
| Run multiple agents that talk to each other | **Agent Team** |
| Connect to an external service (DB, Slack, browser) | **MCP** |
| Run a linter/formatter after every edit automatically | **Hook** |
| Package skills + hooks + MCP for distribution | **Plugin** |

---

## Head-to-Head Comparisons

### Skill vs Subagent

| Aspect | Skill | Subagent |
| --- | --- | --- |
| What it is | Reusable instructions/knowledge/workflows | Isolated worker with its own context |
| Key benefit | Share content across contexts | Context isolation — only summary returns |
| Context impact | Loads into your main session | Completely separate context |
| Best for | Reference material, invocable workflows | Tasks reading many files, parallel work |
| Invocation | `/<name>` or auto-detected | Agent tool or natural language |

> **They combine:** A subagent can preload specific skills via its `skills:` field. A skill can run in isolated context using `context: fork`.

### CLAUDE.md vs Skill

| Aspect | CLAUDE.md | Skill |
| --- | --- | --- |
| Loads | Every session, automatically | On demand |
| Can trigger workflows | No | Yes, with `/<name>` |
| Best for | "Always do X" rules | Reference material, invocable workflows |
| Context cost | Every request | Low until used |

> **Rule of thumb:** Keep CLAUDE.md under 200 lines. Move reference content to skills.

### CLAUDE.md vs Rules vs Skills

| Aspect | CLAUDE.md | `.claude/rules/` | Skill |
| --- | --- | --- | --- |
| Loads | Every session | Every session (or when matching files opened) | On demand |
| Scope | Whole project | Can be scoped to file paths | Task-specific |
| Best for | Core conventions | Language/directory-specific guidelines | Reference material, workflows |

### Subagent vs Agent Team

| Aspect | Subagent | Agent Team |
| --- | --- | --- |
| Context | Own window; results return to caller | Own window; fully independent |
| Communication | Reports back to main agent only | Teammates message each other directly |
| Coordination | Main agent manages all work | Shared task list with self-coordination |
| Best for | Focused tasks where only result matters | Complex work requiring discussion |
| Token cost | Lower: results summarised back | Higher: each teammate is separate instance |

> **Transition point:** If parallel subagents hit context limits or need to communicate with each other, agent teams are the natural next step.

### MCP vs Skill

| Aspect | MCP | Skill |
| --- | --- | --- |
| What it is | Protocol for connecting to external services | Knowledge, workflows, reference material |
| Provides | Tools and data access | Knowledge about how to use those tools |
| Example | Connects Claude to your database | Teaches Claude your data model and query patterns |

> These solve different problems and work well together. **MCP gives Claude the ability; a skill teaches Claude how to use it effectively.**

---

## Feature Loading & Context Costs

### Loading Timeline

```
Session Start
  ├── CLAUDE.md ─────────── Full content loaded (every request)
  ├── MCP servers ────────── All tool definitions loaded (every request)
  ├── Skill descriptions ── Names + descriptions only (low cost)
  └── Auto memory ─────────  First 200 lines of MEMORY.md

During Session
  ├── Skill invoked ──────── Full content loaded into conversation
  ├── Subagent spawned ──── Fresh isolated context created
  └── Hook triggered ─────── External script runs (zero context cost)
```

### How Features Layer (Priority)

When the same feature exists at multiple levels:

| Feature | Layering |
| --- | --- |
| CLAUDE.md | Additive — all levels contribute simultaneously |
| Skills | Override by name: managed > user > project |
| Subagents | Override by name: managed > CLI flag > project > user > plugin |
| MCP servers | Override by name: local > project > user |
| Hooks | Merge — all registered hooks fire for matching events |

---

## Combination Patterns

| Pattern | How It Works | Example |
| --- | --- | --- |
| Skill + MCP | MCP provides connection; skill teaches usage | MCP connects to DB; skill documents schema and query patterns |
| Skill + Subagent | Skill spawns subagents for parallel work | `/audit` kicks off security + performance + style subagents |
| CLAUDE.md + Skills | CLAUDE.md has always-on rules; skills have reference material | CLAUDE.md says "follow API conventions"; skill has the full guide |
| Hook + MCP | Hook triggers external actions through MCP | Post-edit hook sends Slack notification for critical file changes |

---

## Practical Guidance

### When NOT to Use Each Feature

| Feature | Skip If… |
| --- | --- |
| CLAUDE.md | Claude already does it correctly by default |
| Skill | The content is a one-time instruction, not reusable |
| Subagent | The task is quick and doesn't generate verbose output |
| Agent Team | Work is sequential or teammates would edit the same files |
| MCP | You can achieve it with a Bash command |
| Hook | The action only matters sometimes (use CLAUDE.md instead) |

### Context Budget Awareness

- MCP servers add tool definitions to **every request** — a few servers can consume significant context before you start working.
- Run `/mcp` to check per-server token costs.
- Run `/context` to see overall context usage.
- Disconnect MCP servers you're not actively using.
- Use `disable-model-invocation: true` on skills you only trigger manually (zero cost until invoked).