# 11_Slash Commands & Skills

# M08 — Slash Commands & Skills

> 15 built-in commands, the SKILL.md format, custom workflows, legacy commands, and SDK integration.

---

## Built-in Slash Commands

Slash commands are prefixed with `/` and control Claude Code sessions. There are 15 built-in:

| Command | What It Does |
| --- | --- |
| `/compact` | Summarise conversation history to free context |
| `/clear` | Start fresh — clear all previous history |
| `/help` | Get help with Claude Code features |
| `/model` | Switch models mid-session |
| `/hooks` | Browse configured hooks |
| `/agents` | Manage subagents |
| `/permissions` | Configure tool permissions |
| `/mcp` | Manage MCP servers |
| `/context` | See what's consuming context |
| `/init` | Generate a starter CLAUDE.md |
| `/rewind` | Open rewind menu to restore checkpoints |
| `/statusline` | Configure status line |
| `/btw` | Side question that doesn't enter context |
| `/rename` | Rename the current session |
| `/cost` | Show session cost and token usage |

---

## Skills: The Modern Custom Command System

Custom slash commands are defined as **skills** — Markdown files in `.claude/skills/` or `~/.claude/skills/`. Skills extend Claude's knowledge with instructions, workflows, and reference material.

> **Three invocation modes:** manually with `/<name>`, auto-detected by Claude based on the task description, or both (default).

Create a directory with a `SKILL.md` file:

```
.claude/skills/
  └── deploy/
      └── SKILL.md
```

```markdown
---
name: deploy
description: Deploy the application to production
disable-model-invocation: true
---

Deploy the application using the following steps:

1. Run `npm test` to verify all tests pass
2. Run `npm run build` to create production build
3. Run `npm run deploy` to deploy
4. Verify deployment with `curl https://app.example.com/health`
```

Invoke with `/deploy`.

---

## Skill Frontmatter Fields

| Field | Description |
| --- | --- |
| `name` | Command name (used for `/<name>`) |
| `description` | When Claude should use this skill (also shown in typeahead) |
| `disable-model-invocation` | `true` = only manually invoked. Hides from auto-detection (zero context cost) |
| `allowed-tools` | Restrict which tools the skill can use |
| `model` | Override the model for this skill |
| `context` | `fork` to run in isolated subagent context |

### Skill Locations & Priority

| Location | Scope | Priority |
| --- | --- | --- |
| `.claude/skills/` | Project | Higher |
| `~/.claude/skills/` | All projects | Lower |
| Plugin skills | Where plugin enabled | Lowest |

---

## Skill with Arguments

Use `$ARGUMENTS` to capture everything after the command name:

```markdown
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---

Fix issue #$ARGUMENTS:

1. Use `gh issue view $ARGUMENTS` to get issue details
2. Understand the problem
3. Search for relevant files
4. Implement the fix
5. Write and run tests
6. Commit and create a PR
```

Invoke: `/fix-issue 1234`

---

## Legacy Format: `.claude/commands/`

The legacy format uses `.claude/commands/` with plain Markdown files. The filename (minus `.md`) becomes the command name. It supports positional args (`$1`, `$2`).

```markdown
---
argument-hint: [issue-number] [priority]
description: Fix a GitHub issue
---

Fix issue #$1 with priority $2.
Check the issue description and implement the necessary changes.
```

Use: `/fix-issue 123 high` → `$1="123"`, `$2="high"`

**Bash Execution & File References** — include shell output with `!` backtick syntax, and file contents with `@`:

```markdown
---
description: Create a git commit
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)
---

## Context
- Current status: !`git status`
- Current diff: !`git diff HEAD`
- Package config: @package.json
- TypeScript config: @tsconfig.json
```

---

## Skills vs Legacy Commands

| Aspect | Skills (`.claude/skills/`) | Legacy (`.claude/commands/`) |
| --- | --- | --- |
| Format | Directory with `SKILL.md` | Single `.md` file |
| Auto-detection | Yes (Claude can invoke based on task) | No (manual only) |
| Context cost | Description loaded at start; full content on demand | Loaded when invoked |
| Frontmatter | Full skill frontmatter | Limited frontmatter |
| Recommended | **Yes** | Legacy |

> **Migration:** Move `.claude/commands/foo.md` to `.claude/skills/foo/SKILL.md`. Both formats continue to work.

---

## Using Slash Commands via the Agent SDK

**Discovering available commands** — read `message.slash_commands` from the `system`/`init` message:

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({ prompt: "Hello Claude", options: { maxTurns: 1 } })) {
  if (message.type === "system" && message.subtype === "init") {
    console.log("Available commands:", message.slash_commands);
  }
}
```

```python
import asyncio
from claude_agent_sdk import query

async def main():
    async for message in query(prompt="Hello Claude", options={"max_turns": 1}):
        if message.type == "system" and message.subtype == "init":
            print("Available commands:", message.slash_commands)

asyncio.run(main())
```

**Executing commands via SDK** — send the command as the `prompt` (e.g. `/compact`, `/code-review`) and handle system messages like `compact_boundary`.

---

## Practical Skill Examples

**Code Review** — `!`-embeds `git diff --name-only HEAD~1` and `git diff HEAD~1`, then a review checklist (quality, security, performance, tests, docs).

**Test Runner** — takes `$ARGUMENTS` as a pattern; detects the framework, runs matching tests, fixes failures, re-runs.

**Deploy Workflow** — `$ARGUMENTS` selects target (default staging); verifies tests, builds, deploys, runs smoke tests, reports status.