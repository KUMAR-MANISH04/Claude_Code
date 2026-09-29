# 01_Agentic Loop & Tools

# Agentic Loop & Tools

> Understand the core architecture: the three-phase agentic loop, the complete tools reference, context window management, and the extension layer.

---

## Core Architecture

Claude Code is an **agentic harness** around Claude models. It provides tools, context management, and an execution environment that turns a language model into a capable coding agent.

### The Three-Phase Agentic Loop

```
Your Prompt → Gather Context → Take Action → Verify Results → (repeat until done)
                    ↑_______________↓________________↓
                         You can interrupt at any point
```

- **Gather Context** — Read files, search the codebase, run commands to understand the problem.
- **Take Action** — Edit files, run builds, make API calls.
- **Verify Results** — Run tests, check output, compare against expected behaviour.

> **Key insight:** Each tool use gives Claude new information that informs the next step. The loop adapts to the task — a question may only need context gathering; a bug fix cycles through all three phases repeatedly.

---

## Complete Tools Reference

Tool names are the exact strings used in permission rules, subagent tool lists, and hook matchers.

### File Operations

| Tool | Description | Permission |
| --- | --- | --- |
| `Read` | Reads file contents (text, images, PDFs, Jupyter notebooks) | No |
| `Write` | Creates or overwrites files | Yes |
| `Edit` | Targeted edits via string replacement | Yes |
| `NotebookEdit` | Modifies Jupyter notebook cells | Yes |
| `Glob` | Finds files by pattern matching | No |
| `Grep` | Searches for regex patterns in file contents | No |

### Execution

| Tool | Description | Permission |
| --- | --- | --- |
| `Bash` | Executes shell commands in your environment | Yes |

> **Bash behaviour:** Each command runs in a separate process. Working directory persists across commands. Environment variables do **not** — an `export` in one command is lost in the next.

### Web

| Tool | Description | Permission |
| --- | --- | --- |
| `WebFetch` | Fetches content from a specified URL | Yes |
| `WebSearch` | Performs web searches | Yes |

### Code Intelligence

| Tool | Description | Permission |
| --- | --- | --- |
| `LSP` | Type errors, warnings, definitions, references, symbols via language servers (requires a code intelligence plugin) | No |

### Orchestration

| Tool | Description | Permission |
| --- | --- | --- |
| `Agent` | Spawns a subagent with its own context window | No |
| `AskUserQuestion` | Asks multiple-choice questions to gather requirements | No |
| `EnterPlanMode` | Switches to plan mode for read-only exploration | No |
| `ExitPlanMode` | Presents a plan for approval and exits plan mode | Yes |
| `EnterWorktree` | Creates an isolated git worktree | No |
| `ExitWorktree` | Exits a worktree session | No |
| `Skill` | Executes a skill within the main conversation | Yes |

### Task Management

| Tool | Description | Permission |
| --- | --- | --- |
| `TaskCreate` | Creates a new task in the task list | No |
| `TaskGet` | Retrieves full details for a specific task | No |
| `TaskList` | Lists all tasks with their current status | No |
| `TaskUpdate` | Updates task status, dependencies, details | No |
| `TaskOutput` | Retrieves output from a background task | No |
| `TaskStop` | Kills a running background task by ID | No |
| `TodoWrite` | Manages session task checklist (non-interactive / Agent SDK) | No |

### MCP Integration

| Tool | Description | Permission |
| --- | --- | --- |
| `ListMcpResourcesTool` | Lists resources exposed by connected MCP servers | No |
| `ReadMcpResourceTool` | Reads a specific MCP resource by URI | No |
| `ToolSearch` | Searches for and loads deferred tools when tool search is enabled | No |

### Scheduled Tasks

| Tool | Description | Permission |
| --- | --- | --- |
| `CronCreate` | Schedules a recurring or one-shot prompt within the session | No |
| `CronDelete` | Cancels a scheduled task by ID | No |
| `CronList` | Lists all scheduled tasks in the session | No |

---

## Context Window Management

The context window holds: conversation history, file contents, command outputs, CLAUDE.md, auto memory, loaded skills, and system instructions.

**When context fills up:**

1. Claude Code clears older tool outputs first.
2. Then summarises the conversation if needed.
3. Your requests and key code snippets are preserved.
4. Detailed instructions from early in the conversation may be lost.

### Context Cost by Feature

| Feature | When It Loads | Context Cost |
| --- | --- | --- |
| CLAUDE.md | Session start | Every request |
| Skills | Start (descriptions) + when used (full) | Low until used |
| MCP servers | Session start (all tool definitions) | Every request |
| Subagents | When spawned | Isolated from main session |
| Hooks | On trigger | Zero (runs externally) |

> **Key implication:** Context is the most important resource to manage. Every file read, command output, and tool result consumes it — which is exactly why subagents and agent teams exist: they provide context isolation.

---

## Access Model

When Claude Code runs in a directory, it can access:

| Resource | Details |
| --- | --- |
| Project files | Files in your directory and subdirectories |
| Terminal | Any command you could run: build tools, git, package managers |
| Git state | Current branch, uncommitted changes, recent history |
| CLAUDE.md | Persistent project-specific instructions |
| Auto memory | Learnings saved automatically (patterns, preferences) |
| Extensions | MCP servers, skills, subagents, plugins |

---

## Extension Layer

Extensions plug into different parts of the agentic loop:

| Feature | What It Does | When to Use |
| --- | --- | --- |
| CLAUDE.md | Persistent context loaded every session | "Always do X" rules, conventions |
| Skill | Instructions/knowledge Claude can load | Reference docs, repeatable workflows |
| Subagent | Isolated execution returning summaries | Context isolation, parallel tasks |
| Agent Team | Coordinate multiple independent sessions | Parallel research, collaborative debugging |
| MCP | Connect to external services | External data or actions |
| Hook | Deterministic script on events | Automation without LLM involvement |
| Plugin | Bundle all the above | Reuse across repos, distribute to teams |

---

## Sessions

- Each session starts with a **fresh context window** (no history from previous sessions).
- Sessions are tied to directories — resume only shows sessions from the current directory.
- `claude --continue` resumes the most recent; `claude --resume` selects from recent.
- `claude --continue --fork-session` branches from a session without affecting the original.
- Persistent context across sessions uses **auto memory** and **CLAUDE.md**.

---

## Permission Modes

| Mode | Behaviour |
| --- | --- |
| Default | Claude asks before file edits and shell commands |
| Auto-accept edits | Claude edits files freely, still asks for commands |
| Plan mode | Read-only tools only, creates a plan for approval |

> Cycle with **Shift+Tab**. Configure allowlists in `.claude/settings.json`.