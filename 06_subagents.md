# Subagents

> Fundamentals, built-in subagents, YAML frontmatter fields, advanced configuration (MCP, skills, hooks, memory, worktrees), 8 example templates, and SDK integration.

---

## What Are Subagents?

Subagents are specialised AI assistants that handle specific types of tasks. Each runs in its **own context window** with a custom system prompt, specific tool access, independent permissions, and its own model selection.

> **Key constraint:** Subagents cannot spawn other subagents. If your workflow requires nested delegation, use skills or chain subagents from the main conversation.

### Why Subagents Exist

| Problem | How Subagents Solve It |
| --- | --- |
| Context window fills with exploration output | Subagent explores in isolation; only summary returns |
| Need different permissions for different tasks | Each subagent has scoped tool access |
| Want to reuse configurations across projects | User-level subagents available everywhere |
| Need specialised behaviour for a domain | Focused system prompts for specific work |
| Cost optimisation | Route tasks to cheaper/faster models like Haiku |

---

## Built-in Subagents

Shipped with Claude Code and used automatically when appropriate:

| Agent | Model | Tools | Purpose |
| --- | --- | --- | --- |
| Explore | Haiku (fast) | Read-only (denied Write, Edit) | File discovery, code search, codebase exploration |
| Plan | Inherits | Read-only | Codebase research during plan mode |
| General-purpose | Inherits | All tools | Complex research, multi-step operations |
| Bash | Inherits | Bash | Running terminal commands in separate context |
| Claude Code Guide | Haiku | — | When you ask about Claude Code features |

---

## What Subagents Inherit

| Subagent Receives | Subagent Does NOT Receive |
| --- | --- |
| Its own system prompt (markdown body or `AgentDefinition.prompt`) | Parent's conversation history or tool results |
| The Agent tool's prompt string (from parent) | Parent's system prompt |
| Project CLAUDE.md (loaded normally) | Skills (unless listed in `skills:` field) |
| Tool definitions (inherited or scoped subset) | Auto memory from parent session |

> **Key implication:** The only channel from parent to subagent is the Agent tool's prompt string. Include any file paths, error messages, or decisions the subagent needs directly in that prompt.

---

## All Frontmatter Fields

Subagents are Markdown files with YAML frontmatter. The frontmatter defines metadata; the body becomes the system prompt.

| Field | Required | Description |
| --- | --- | --- |
| `name` | Yes | Unique identifier (lowercase letters and hyphens) |
| `description` | Yes | When Claude should delegate to this subagent |
| `tools` | No | Tools the subagent can use (inherits all if omitted) |
| `disallowedTools` | No | Tools to deny (removed from inherited list) |
| `model` | No | `sonnet`, `opus`, `haiku`, full model ID, or `inherit` (default) |
| `permissionMode` | No | `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan` |
| `maxTurns` | No | Maximum agentic turns before the subagent stops |
| `skills` | No | Skills to preload into subagent's context at startup |
| `mcpServers` | No | MCP servers available to this subagent |
| `hooks` | No | Lifecycle hooks scoped to this subagent |
| `memory` | No | Persistent memory scope: `user`, `project`, or `local` |
| `background` | No | `true` to always run as background task |
| `effort` | No | `low`, `medium`, `high`, `max` (Opus 4.6 only) |
| `isolation` | No | `worktree` for isolated git worktree |

### Scope & Priority

| Location | Scope | Priority |
| --- | --- | --- |
| `--agents` CLI flag | Current session | 1 (highest) |
| `.claude/agents/` | Current project | 2 |
| `~/.claude/agents/` | All your projects | 3 |
| Plugin's `agents/` directory | Where plugin enabled | 4 (lowest) |

---

## Tool Access Control

**Allowlist (`tools`)**

```yaml
---
name: safe-researcher
description: Research agent with restricted capabilities
tools: Read, Grep, Glob, Bash
---
```

**Denylist (`disallowedTools`)**

```yaml
---
name: no-writes
description: Inherits every tool except file writes
disallowedTools: Write, Edit
---
```

> If both are set: `disallowedTools` applied first, then `tools` resolved against the remaining pool.

**Restrict which subagents can be spawned**

```yaml
---
name: coordinator
description: Coordinates work across specialised agents
tools: Agent(worker, researcher), Read, Bash
---
```

> Only `worker` and `researcher` can be spawned. `Agent` without parentheses allows any subagent.

---
## Advanced Configuration

### Scoping MCP Servers

```yaml
---
name: browser-tester
description: Tests features in a real browser using Playwright
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github
---

Use the Playwright tools to navigate, screenshot, and interact with pages.
```

> **Key insight:** Define MCP servers inline in a subagent to keep their tool descriptions OUT of the main conversation context.

### Preloading Skills

```yaml
---
name: api-developer
description: Implement API endpoints following team conventions
skills:
  - api-conventions
  - error-handling-patterns
---

Implement API endpoints. Follow the conventions from the preloaded skills.
```

> Full content is injected, not just made available. Subagents do NOT inherit skills from the parent — you must list them explicitly.

### Persistent Memory

| Scope | Location | Use When |
| --- | --- | --- |
| `user` | `~/.claude/agent-memory/<name>/` | Learnings apply across all projects |
| `project` | `.claude/agent-memory/<name>/` | Project-specific, shareable via VCS |
| `local` | `.claude/agent-memory-local/<name>/` | Project-specific, not committed |

### Hooks in Frontmatter

```yaml
---
name: code-reviewer
description: Review code changes with automatic linting
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh $TOOL_INPUT"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
---
```

### Git Worktree Isolation

```yaml
---
name: experimental-refactor
description: Try risky refactoring in isolation
isolation: worktree
---
```

> Creates an isolated copy of the repository. Automatically cleaned up if the subagent makes no changes.

---

## Invocation Methods

| Method | Example |
| --- | --- |
| Automatic | Claude uses the `description` field to decide when to delegate |
| Natural language | Use the test-runner subagent to fix failing tests |
| @-Mention (guaranteed) | `@"code-reviewer (agent)"` look at the auth changes |
| Session-wide (`--agent`) | `claude --agent code-reviewer` |

> With `--agent`, the subagent's system prompt replaces the default Claude Code system prompt entirely. CLAUDE.md and project memory still load normally.

### Foreground vs Background Execution

| Mode | Behaviour |
| --- | --- |
| Foreground | Blocks main conversation. Permission prompts and AskUserQuestion pass through to you |
| Background | Runs concurrently. Pre-approved permissions only. Clarifying questions fail but subagent continues |

> Claude decides automatically. You can also ask "run this in the background", press **Ctrl+B** to background a running task, or set `background: true` in frontmatter.

---
## Example Templates

### 1. Code Reviewer (Read-Only)

```markdown
---
name: code-reviewer
description: Expert code review specialist. Proactively reviews code for quality, security, and maintainability. Use immediately after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer ensuring high standards of code quality and security.

When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Begin review immediately

Review checklist:
- Code is clear and readable
- No duplicated code
- Proper error handling
- No exposed secrets or API keys
- Input validation implemented

Provide feedback organised by priority:
- Critical issues (must fix)
- Warnings (should fix)
- Suggestions (consider improving)
```

### 2. Debugger (Read + Write)

```markdown
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behaviour. Use proactively when encountering any issues.
tools: Read, Edit, Bash, Grep, Glob
---

You are an expert debugger specialising in root cause analysis.

When invoked:
1. Capture error message and stack trace
2. Identify reproduction steps
3. Isolate the failure location
4. Implement minimal fix
5. Verify solution works

Focus on fixing the underlying issue, not the symptoms.
```

### 3. Security Reviewer (Opus-Powered)

```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior security engineer. Review code for:
- Injection vulnerabilities (SQL, XSS, command injection)
- Authentication and authorisation flaws
- Secrets or credentials in code
- Insecure data handling

Provide specific line references and suggested fixes.
```

### 4. Database Query Validator (Hook-Guarded)

```markdown
---
name: db-reader
description: Execute read-only database queries
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---

You are a database analyst with read-only access.
Execute SELECT queries to answer questions about the data.
```

### 5. API Developer with Skills

```markdown
---
name: api-developer
description: Implement API endpoints following team conventions
skills:
  - api-conventions
  - error-handling-patterns
model: sonnet
---

Implement API endpoints. Follow the conventions from the preloaded skills.
```

### 6. Browser Tester with Scoped MCP

```markdown
---
name: browser-tester
description: Tests features in a real browser using Playwright
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
---

Use the Playwright tools to navigate, screenshot, and interact with pages.
```

### 7. Learning Reviewer with Persistent Memory

```markdown
---
name: pattern-reviewer
description: Reviews code and learns recurring patterns over time
tools: Read, Grep, Glob, Bash
memory: project
---

Before starting a review:
1. Check your memory for known patterns and past issues
2. Apply those learnings to the current review

After completing a review:
1. Note any new patterns or recurring issues
2. Save useful findings to your memory for future reviews
```

### 8. Data Scientist (Domain-Specific)

```markdown
---
name: data-scientist
description: Data analysis expert for SQL queries and data insights
tools: Bash, Read, Write
model: sonnet
---

You are a data scientist specialising in SQL and BigQuery analysis.
Write efficient SQL queries. Analyse and summarise results.
Always ensure queries are efficient and cost-effective.
```

---

## SDK Integration

The Claude Agent SDK provides programmatic control over subagents via the `agents` parameter in `query()`.

### Python

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def main():
    async for message in query(
        prompt="Review the authentication module for security issues",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Grep", "Glob", "Agent"],
            agents={
                "code-reviewer": AgentDefinition(
                    description="Expert code review specialist.",
                    prompt="You are a code review specialist with expertise in security...",
                    tools=["Read", "Grep", "Glob"],
                    model="sonnet",
                ),
            },
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

### TypeScript

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Review the authentication module for security issues",
  options: {
    allowedTools: ["Read", "Grep", "Glob", "Agent"],
    agents: {
      "code-reviewer": {
        description: "Expert code review specialist.",
        prompt: `You are a code review specialist...`,
        tools: ["Read", "Grep", "Glob"],
        model: "sonnet"
      }
    }
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

### AgentDefinition Fields (SDK)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `description` | string | Yes | When Claude should delegate to this agent |
| `prompt` | string | Yes | System prompt defining role and behaviour |
| `tools` | string[] | No | Allowed tools. Inherits all if omitted |
| `model` | `'sonnet' \| 'opus' \| 'haiku' \| 'inherit'` | No | Model override. Defaults to main model |

> **Important:** Subagents cannot spawn their own subagents. Do NOT include `Agent` in a subagent's `tools` array.

### Detecting Subagent Invocation

Check for `tool_use` blocks where `name` is `"Agent"`. Messages from within a subagent include `parent_tool_use_id`.

> **Note:** The tool was renamed from `"Task"` to `"Agent"` in v2.1.63. Check both values for cross-version compatibility.