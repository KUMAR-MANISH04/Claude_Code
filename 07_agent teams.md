# 07_agent teams

# Agent Teams (Experimental)

> Architecture, task management, communication, display modes, use cases, team sizing best practices, and known limitations.

---

## What Are Agent Teams?

Agent teams coordinate multiple Claude Code instances working together. One session acts as the **team lead**, coordinating work and assigning tasks. **Teammates** work independently, each in its own context window, and communicate directly with each other.

> **Status:** Agent teams are experimental and disabled by default. Requires Claude Code v2.1.32+.

### Subagents vs Agent Teams

| | Subagents | Agent Teams |
| --- | --- | --- |
| Context | Own window; results return to caller | Own window; fully independent |
| Communication | Report back to main agent only | Teammates message each other directly |
| Coordination | Main agent manages all work | Shared task list with self-coordination |
| Best for | Focused tasks where only result matters | Complex work requiring discussion |
| Token cost | Lower: results summarised back | Higher: each is a separate instance |
| You can talk to | Main agent only | Any teammate directly |

---

## Architecture

| Component | Role |
| --- | --- |
| Team Lead | Main Claude Code session that creates the team, spawns teammates, coordinates work |
| Teammates | Separate Claude Code instances working on assigned tasks |
| Task List | Shared list of work items that teammates claim and complete |
| Mailbox | Messaging system for communication between agents |

> Team config is stored at `~/.claude/teams/{team-name}/config.json`. Task lists at `~/.claude/tasks/{team-name}/`.

### Enabling Agent Teams

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Or in your shell: `export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

---

## When to Use Agent Teams

### Strong Use Cases

| Pattern | Why It Works |
| --- | --- |
| Research and review | Multiple teammates investigate different aspects simultaneously, share and challenge findings |
| New modules or features | Each teammate owns a separate piece without stepping on each other |
| Debugging with competing hypotheses | Teammates test different theories in parallel, converge faster |
| Cross-layer coordination | Frontend, backend, and test changes each owned by a different teammate |

### When NOT to Use

> Agent teams add coordination overhead and use significantly more tokens. Avoid when:

- Work is sequential
- Tasks involve editing the same files
- Work has many dependencies between tasks
- A single session or subagents would suffice

---

## Display Modes

| Mode | Description | Requirement |
| --- | --- | --- |
| In-process | All teammates run inside main terminal. **Shift+Down** cycles between them | Any terminal |
| Split panes | Each teammate gets its own pane. See everyone at once | tmux or iTerm2 |
| Auto (default) | Split panes if already in tmux, otherwise in-process | — |

### In-Process Controls

| Action | Key |
| --- | --- |
| Cycle through teammates | Shift+Down |
| View teammate session | Enter |
| Interrupt teammate's turn | Escape |
| Toggle task list | Ctrl+T |

---

## Task Management

The shared task list coordinates work. Tasks have three states: **pending**, **in progress**, **completed**. Tasks can depend on other tasks — a pending task with unresolved dependencies cannot be claimed until dependencies complete.

### Task Assignment

| Method | How |
| --- | --- |
| Lead assigns | Tell the lead which task to give to which teammate |
| Self-claim | After finishing a task, teammate picks up next unassigned, unblocked task |

---

## Communication

- **Automatic delivery:** Messages from teammates delivered as new conversation turns.
- **Queued mid-turn:** If you're mid-turn, messages are queued and delivered when the turn ends.
- **Idle state:** Teammates go idle after every turn — completely normal. Sending a message wakes them up.
- **Direct messaging:** In-process — Shift+Down to cycle, then type. Split-pane — click into teammate's pane.

---

## Use Case Patterns

### Parallel Code Review

```text
Create an agent team to review PR #142. Spawn three reviewers:
- One focused on security implications
- One checking performance impact
- One validating test coverage
Have them each review and report findings.
```

Each reviewer applies a different filter. The lead synthesises findings across all three after they finish.

### Competing Hypotheses Debugging

```text
Users report the app exits after one message instead of staying connected.
Spawn 5 agent teammates to investigate different hypotheses. Have them
talk to each other to try to disprove each other's theories, like a
scientific debate.
```

The debate structure is the key mechanism. Adversarial parallel investigation finds the strongest surviving theory.

### Cross-Layer Feature Development

```text
Create an agent team for the new user profile feature:
- Frontend teammate: implement the profile page UI
- Backend teammate: create the API endpoints
- Test teammate: write integration tests (depends on both above)
```

Each teammate has deep context for their layer without the noise of other layers.

### Parallel Research

```text
Create an agent team to understand this legacy codebase:
- One teammate maps the authentication system
- One teammate maps the data layer
- One teammate maps the API surface
Have them share their findings when done.
```

---

## Best Practices

### Team Sizing

| Guideline | Recommendation |
| --- | --- |
| Starting point | 3–5 teammates |
| Tasks per teammate | 5–6 keeps everyone productive |
| Scaling rule | Only add teammates when work genuinely benefits from parallelism |
| Diminishing returns | 3 focused teammates often outperform 5 scattered ones |

> Token costs scale linearly. Coordination overhead increases non-linearly.

### Practical Tips

- **Give enough context:** Teammates don't see the lead's conversation history — include task-specific details in the spawn prompt.
- **Prevent file conflicts:** Break work so each teammate owns different files.
- **Monitor and steer:** Check in on progress, redirect approaches that aren't working.
- **Handle the "eager lead":** If the lead starts implementing itself, tell it to wait for teammates.
- **Start with research/review:** If new to agent teams, begin with non-coding tasks that have clear boundaries.

### Quality Gates with Hooks

| Hook Event | Matcher Input | When |
| --- | --- | --- |
| TeammateIdle | — | Teammate about to go idle. Exit 2 sends feedback and keeps them working |
| TaskCompleted | — | Task being marked complete. Exit 2 prevents completion and sends feedback |

---

## Decision Tree

```
Need parallel work?
├── No → Use single session
└── Yes
    ├── Workers need to talk to each other?
    │   ├── No → Use subagents (lower token cost)
    │   └── Yes → Use agent teams
    ├── Hitting context limits with subagents?
    │   └── Yes → Use agent teams (fully independent contexts)
    ├── Need manual parallel sessions?
    │   └── Use git worktrees
    └── Work involves same files?
        └── Yes → Single session (avoid conflicts)
```

---

## Known Limitations

| Limitation | Details |
| --- | --- |
| No session resumption | `/resume` and `/rewind` don't restore in-process teammates |
| Task status lag | Teammates sometimes fail to mark tasks complete; check manually |
| Slow shutdown | Teammates finish current request before shutting down |
| One team per session | Clean up current team before starting a new one |
| No nested teams | Teammates cannot spawn their own teams |
| Lead is fixed | Can't promote a teammate or transfer leadership |
| Permissions at spawn | All teammates start with lead's mode; change individually after |
| Split panes | Not supported in VS Code terminal, Windows Terminal, or Ghostty |