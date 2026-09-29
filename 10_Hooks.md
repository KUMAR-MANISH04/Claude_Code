# M07 — Hooks

> 22 hook events, 4 hook types, exit code semantics, matcher patterns, configuration locations, and 15 ready-to-use patterns.

---

## What Are Hooks?

Hooks are user-defined commands that execute automatically at specific points in Claude Code's lifecycle. They provide **deterministic** control — unlike CLAUDE.md instructions which are advisory, hooks **guarantee** the action happens.

### Four Hook Types

| Type | What It Does | Use When |
| --- | --- | --- |
| `command` | Runs a shell command | Most common. JSON stdin, stdout/stderr/exit code output |
| `http` | POSTs event data to a URL | External services, shared audit, cloud functions |
| `prompt` | Single-turn LLM evaluation (yes/no) | Judgment-based decisions (is this safe?) |
| `agent` | Spawns a subagent with tool access | Verification requiring file inspection or test execution |

### Hooks vs Other Features

| Feature | When to Use |
| --- | --- |
| **Hooks** | "This MUST happen every time" — deterministic, no LLM involved |
| **CLAUDE.md** | "Claude SHOULD do this" — advisory, LLM decides |
| **Skills** | "Claude CAN do this" — on-demand knowledge/workflows |
| **Subagents** | "Do this in isolation" — context management |

---

## Configuration

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "regex_pattern",
        "hooks": [
          {
            "type": "command",
            "command": "path/to/script.sh",
            "timeout": 600,
            "statusMessage": "Custom message shown during execution"
          }
        ]
      }
    ]
  }
}
```

### Configuration Locations

| Location | Scope | Shareable |
| --- | --- | --- |
| `~/.claude/settings.json` | All your projects | No (local) |
| `.claude/settings.json` | Single project | Yes (commit) |
| `.claude/settings.local.json` | Single project | No (gitignored) |
| Managed policy settings | Organisation-wide | Yes (admin) |
| Plugin `hooks/hooks.json` | When plugin enabled | Yes (bundled) |
| Skill/agent frontmatter | While component active | Yes (in file) |

---

## Exit Code Semantics

| Exit Code | Meaning | Effect |
| --- | --- | --- |
| `0` | Success | Action proceeds. Parse stdout for JSON |
| `2` | Block | Action blocked. stderr fed back to Claude as feedback |
| Other | Non-blocking error | Logged in verbose mode. Action continues |

### Which Events Can Block?

- **Can block (exit 2):** `PreToolUse`, `PermissionRequest`, `UserPromptSubmit`, `Stop`, `SubagentStop`, `TaskCompleted`, `TeammateIdle`, `ConfigChange`, `Elicitation`, `ElicitationResult`, `WorktreeCreate`
- **Cannot block:** `PostToolUse`, `PostToolUseFailure`, `Notification`, `SubagentStart`, `SessionStart`, `SessionEnd`, `PreCompact`, `PostCompact`, `InstructionsLoaded`, `WorktreeRemove`, `StopFailure`

---

## Matcher Patterns

Matchers are regex patterns that filter when hooks fire. Without a matcher, the hook fires on every occurrence.

| Event(s) | Matches Against | Examples |
| --- | --- | --- |
| `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest` | Tool name | `Bash`, `Edit\|Write`, `mcp__.*` |
| `SessionStart` | Session source | `startup`, `resume`, `clear`, `compact` |
| `SessionEnd` | Exit reason | `clear`, `resume`, `logout` |
| `Notification` | Notification type | `permission_prompt`, `idle_prompt` |
| `SubagentStart`, `SubagentStop` | Agent type | `Explore`, `Bash`, custom names |
| `PreCompact`, `PostCompact` | Trigger type | `manual`, `auto` |

> MCP tools use the format `mcp__<server>__<tool>`, e.g. `mcp__github__.*`.

---
## All 22 Hook Events

**Session Lifecycle**

| Event | When | Can Block | Key Input |
| --- | --- | --- | --- |
| `SessionStart` | Session begins or resumes | No | `source`, `model`, `$CLAUDE_ENV_FILE` |
| `SessionEnd` | Session terminates | No | `reason` |

**User Input**

| Event | When | Can Block | Key Input |
| --- | --- | --- | --- |
| `UserPromptSubmit` | User submits a prompt | Yes | `prompt` |

**Tool Lifecycle**

| Event | When | Can Block | Key Input |
| --- | --- | --- | --- |
| `PreToolUse` | Before a tool call executes | Yes | `tool_name`, `tool_input` |
| `PermissionRequest` | Permission dialog about to show | Yes | `permission_suggestions` |
| `PostToolUse` | After a tool call succeeds | No | `tool_response` |
| `PostToolUseFailure` | After a tool call fails | No | `error`, `is_interrupt` |

**Agent Lifecycle**

| Event | When | Can Block | Key Input |
| --- | --- | --- | --- |
| `SubagentStart` | Subagent spawned | No | `agent_id`, `agent_type` |
| `SubagentStop` | Subagent finishes | Yes | `last_assistant_message` |
| `Stop` | Main agent finishes responding | Yes | `stop_hook_active`, `last_assistant_message` |
| `StopFailure` | Turn ends due to API error | No | Error type matcher |

**Agent Teams**

| Event | When | Can Block | Key Input |
| --- | --- | --- | --- |
| `TeammateIdle` | Teammate about to go idle | Yes | `teammate_name`, `team_name` |
| `TaskCompleted` | Task marked complete | Yes | `task_id`, `task_subject` |

**Configuration & Context**

| Event | When | Can Block | Key Input |
| --- | --- | --- | --- |
| `InstructionsLoaded` | CLAUDE.md or rules loaded | No | `file_path`, `load_reason` |
| `ConfigChange` | Config file changes during session | Yes* | Config source |
| `Notification` | Claude Code sends a notification | No | Notification type |
| `PreCompact` | Before context compaction | No | `trigger`, `custom_instructions` |
| `PostCompact` | After context compaction | No | `compact_summary` |

**Worktrees & MCP Elicitation**

| Event | When | Can Block | Key Input |
| --- | --- | --- | --- |
| `WorktreeCreate` | Creating isolated worktree | Yes | Print worktree path to stdout |
| `WorktreeRemove` | Removing worktree | No | Cleanup only |
| `Elicitation` | MCP server requests user input | Yes | `message`, `requested_schema` |
| `ElicitationResult` | After user responds to MCP elicitation | Yes | MCP server name |

---

## Hook Input & Output

**Input (stdin)** — Every hook receives JSON on stdin with common fields:

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "agent_id": "subagent-id",
  "agent_type": "AgentName"
}
```

**Structured Output (JSON)**

```json
{
  "continue": true,
  "decision": "block",
  "reason": "Explanation",
  "additionalContext": "Context injected for Claude",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "Why",
    "updatedInput": { "field": "modified value" }
  }
}
```

> Use exit 2 to block with stderr, OR exit 0 with JSON for structured control. Don't mix them — JSON is ignored on exit 2.

---

## Environment Variables

| Variable | Available In | Description |
| --- | --- | --- |
| `$CLAUDE_PROJECT_DIR` | All hooks | Project root directory |
| `$CLAUDE_ENV_FILE` | SessionStart only | Path to file for persisting env vars |
| `${CLAUDE_PLUGIN_ROOT}` | Plugin hooks | Plugin installation directory |
| `${CLAUDE_PLUGIN_DATA}` | Plugin hooks | Plugin persistent data directory |
| `$CLAUDE_CODE_REMOTE` | All hooks | `"true"` in remote environments |

---
## Ready-to-Use Patterns

**1. Desktop Notifications** — `Notification` hook running `osascript -e 'display notification ...'` (macOS).

**2. Auto-Format After Edits**

```json
{ "hooks": { "PostToolUse": [ { "matcher": "Edit|Write",
  "hooks": [ { "type": "command",
    "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" } ] } ] } }
```

**3. Block Edits to Protected Files**

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
PROTECTED_PATTERNS=(".env" "package-lock.json" ".git/")
for pattern in "${PROTECTED_PATTERNS[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: $FILE_PATH matches protected pattern '$pattern'" >&2
    exit 2
  fi
done
exit 0
```

**4. Re-Inject Context After Compaction** — `SessionStart` with matcher `compact` that echoes reminders (e.g. "use Bun, not npm").

**5. Block Destructive Commands**

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')
if echo "$COMMAND" | grep -qE 'rm -rf|drop table|truncate'; then
  jq -n '{ "hookSpecificOutput": { "hookEventName": "PreToolUse",
    "permissionDecision": "deny", "permissionDecisionReason": "Destructive command blocked." } }'
else
  exit 0
fi
```

**6. Auto-Approve Safe Permissions** — `PermissionRequest` with a narrow matcher (e.g. `ExitPlanMode`) echoing an allow decision.

> **Warning:** Keep matchers narrow. Matching `.*` would auto-approve EVERY permission prompt.

**7. Audit Configuration Changes** — `ConfigChange` hook appending `{timestamp, source, file}` to an audit log.

**8. Log All Bash Commands** — `PostToolUse` matcher `Bash` appending `.tool_input.command` to a log file.

**9. Prompt-Based Hook (LLM Judgment)** — `Stop` hook of type `prompt` that checks if all tasks are complete.

**10. Agent-Based Hook (Multi-Turn Verification)**

```json
{ "hooks": { "Stop": [ { "hooks": [ { "type": "agent",
  "prompt": "Verify that all unit tests pass. Run the test suite and check the results. $ARGUMENTS",
  "timeout": 120 } ] } ] } }
```

> Agent hooks spawn a subagent with Read, Grep, Glob tools and up to 50 tool-use turns.

**11. Quality Gates on Task Completion** — `TaskCompleted` script running `npm test` and `npm run lint`, exit 2 on failure.

**12. HTTP Hook for External Services** — `type: http` POST with `headers` and `allowedEnvVars` for tokens.

**13. TypeScript Validation After Edit** — PostToolUse script running `npx tsc --noEmit` on `.ts` files.

**14. Hooks in Subagent Frontmatter**

```yaml
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
```

**15. Prevent Infinite Stop Loops**

```bash
#!/bin/bash
INPUT=$(cat)
if [ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ]; then
  exit 0  # Allow Claude to stop (prevents infinite loop)
fi
if ! npm test 2>&1; then
  echo "Tests must pass before stopping" >&2
  exit 2
fi
exit 0
```

> **Critical:** Always check `stop_hook_active` in Stop hooks to prevent infinite loops.

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| JSON validation failed | Shell profile (`~/.zshrc`) may output text that corrupts JSON. Guard with `if [[ $- == *i* ]]` |
| Hook not firing | Run `/hooks` to confirm the hook appears. Check matcher case-sensitivity |
| `PermissionRequest` not firing | Doesn't fire in non-interactive mode (`-p`). Use `PreToolUse` instead |
| Command not found | Use absolute paths or `$CLAUDE_PROJECT_DIR` |
| Debugging | `Ctrl+O` toggle verbose mode, `claude --debug` for full details, `/hooks` to browse |