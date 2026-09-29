# 08_Design patterns

# Design Patterns & Best Practices

> The 10 official best practices, 7 multi-agent design patterns, 10 anti-patterns to avoid, and context management strategies.

---

## Official Best Practices

> Source: code.claude.com/docs/en/best-practices.

**The core constraint:** the context window holds your full conversation. Every message, file read, and command output takes space. Claude makes more mistakes as the window fills. Context is the resource you must control first. Run `/context` to see what uses the space.

### 1. Give Claude a check it can run

Claude stops when work looks complete. Without a check, "looks complete" is the only signal and you become the test loop. Give Claude something that returns pass/fail — a test suite, a build exit code, a linter, a fixture comparison, or a screenshot compared with a design.

| Strategy | Before | After |
| --- | --- | --- |
| State the acceptance criteria | "implement a function that validates email addresses" | "write a validateEmail function. Test cases: user@example.com → true, invalid → false, user@.com → false. Run the tests after you implement it" |
| Check UI changes visually | "make the dashboard look better" | "[paste screenshot] implement this design. Take a screenshot of the result and compare it with the original. List the differences and correct them" |
| Ask for the root cause | "the build is failing" | "the build fails with this error: [paste]. Correct it and confirm the build succeeds. Find the root cause. Do not suppress the error" |

**How strictly the check gates the turn:**

| Level | How you set it | Use it when |
| --- | --- | --- |
| In one prompt | Ask Claude to run the check and iterate in the same message | You watch the session |
| Across a session | Set the check as a `/goal` condition; a separate evaluator re-checks after every turn | The task needs many turns |
| As a hard gate | A Stop hook runs your script and blocks the end of the turn until it passes | The check must run every time |
| By a second opinion | A verification subagent or dynamic workflow tries to disprove the result | The agent that did the work must not grade the work |

> **Limit:** Claude Code overrides a Stop hook and ends the turn after 8 consecutive blocks. A hook cannot hold a session open indefinitely.

> **Ask for evidence, not a claim.** Tell Claude to show the test output, the command it ran and the result, or a screenshot.

### 2. Explore, then plan, then code

If Claude writes code immediately, it can solve the wrong problem. Use plan mode to separate research from execution.

- **Explore (plan mode):** press **Shift+Tab** until the status bar shows ⏸ plan mode on, or start with `claude --permission-mode plan`. Claude reads and answers, makes no changes.
- **Plan (plan mode):** ask for a detailed implementation plan. Press **Ctrl+G** to open the plan in your editor and change it.
- **Implement:** approve the plan or press Shift+Tab. Claude writes the code and checks it against the plan.
- **Commit:** ask Claude to commit with a descriptive message and open a pull request.

> **When to skip the plan:** the scope is clear and the change is small (a typo, a log line, a rename). If you can describe the diff in one sentence, do not plan.

### 3. Give specific context in your prompt

Name the files, state the constraints, and point to an example pattern.

| Strategy | Before | After |
| --- | --- | --- |
| Scope the task | "add tests for foo.py" | "write a test for foo.py that covers the edge case where the user is logged out. Do not use mocks" |
| Point to the source | "why does ExecutionFactory have such a weird api?" | "look through the git history of ExecutionFactory and summarise how its api came to be" |
| Reference an existing pattern | "add a calendar widget" | "look at how the existing widgets on the home page are implemented. HotDogWidget.php is a good example. Follow that pattern. Use only the libraries already in the codebase" |
| Describe the symptom | "fix the login bug" | "users report that login fails after session timeout. Check the auth flow in src/auth/, especially token refresh. Write a failing test that reproduces the problem, then correct it" |

**Give Claude rich content:**

- Reference files with `@` instead of describing where the code is. Claude reads the file before answering.
- Paste images directly (copy/paste or drag and drop).
- Give URLs for docs and API references. Use `/permissions` to allowlist domains you use often.
- Pipe data in: `cat error.log | claude`.
- Let Claude collect context with Bash commands, MCP tools, or file reads.

### 4. Ask questions about the codebase

Ask Claude the same questions you'd ask a senior engineer — no special format needed.

- "How does logging work?"
- "How do I make a new API endpoint?"
- "What edge cases does CustomerOnboardingFlowImpl handle?"
- "Why does this code call foo() instead of bar() on line 333?"

### 5. Let Claude interview you

For a large feature, let Claude collect requirements first.

```text
I want to build [brief description]. Interview me in detail using the
AskUserQuestion tool.

Ask about technical implementation, UI/UX, edge cases, concerns, and
tradeoffs. Don't ask obvious questions, dig into the hard parts I
might not have considered.

Keep interviewing until we've covered everything, then write a
complete spec to SPEC.md.
```

Then start a fresh session to execute the spec. A good spec is self-contained: it names files and interfaces, states what is out of scope, and ends with an end-to-end step that proves the feature works.

### 6. Configure your environment

**Write an effective CLAUDE.md** — Run `/init` to generate a starter, then refine. For each line ask: "if I remove this, will Claude make a mistake?" If no, delete it.

| ✅ Include in CLAUDE.md | ❌ Exclude from CLAUDE.md |
| --- | --- |
| Bash commands that Claude cannot guess | Anything Claude can find in the code |
| Code style rules that differ from defaults | Standard language conventions Claude knows |
| Test instructions and preferred test runner | Detailed API documentation (link to it) |
| Repository etiquette (branch names, PR conventions) | Information that changes often |
| Architectural decisions specific to your project | Long explanations or tutorials |
| Environment quirks (required environment variables) | File-by-file descriptions of the codebase |
| Known traps and non-obvious behaviour | Self-evident rules such as "write clean code" |

> Add emphasis such as **IMPORTANT** or **YOU MUST** for rules Claude must not miss. Commit the file to git. Use `@path/to/import` to import another file. Content that applies only sometimes belongs in a skill.

**Configure permissions** — pre-approve trusted tools with `/permissions`, and let sandboxed commands run without a prompt with `/sandbox`.

| Mode | Behaviour | Default for |
| --- | --- | --- |
| Auto | A separate classifier model reviews most actions, blocking only what looks risky (scope escalation, unknown infrastructure, hostile content) | Pro, Max, and Team plans, in interactive terminal and VS Code |
| Manual | Claude Code asks before every action that can modify your system | Other plans |

**Use CLI tools** — CLI tools are the most context-efficient way to reach an external service. Install `gh` for GitHub. Claude can also learn a new tool: `Use 'foo-cli-tool --help' to learn about foo tool, then use it to solve A, B, C`.

**Extend Claude Code:**

| Feature | Use it for | How you add it | Module |
| --- | --- | --- | --- |
| Skills | Domain knowledge and repeatable workflows, loaded on demand | `SKILL.md` in `.claude/skills/` | M08 |
| Subagents | Isolated tasks that read many files | A markdown file in `.claude/agents/` | M04 |
| Hooks | Actions that must happen every time | `.claude/settings.json`, or run `/hooks` | M07 |
| MCP servers | External systems: issue trackers, databases, Figma | `claude mcp add` | M13 |
| Plugins | Bundles of skills, hooks, subagents, and MCP servers | Run `/plugin` | M14 |

> **Instruction or guarantee:** a CLAUDE.md instruction is advisory; a hook is deterministic and always runs. If an action must never be missed, write a hook.

### 7. Control your session

Correct Claude as soon as it goes off track. Tight feedback loops beat long corrections at the end.

| Action | Command |
| --- | --- |
| Stop Claude mid-action and redirect (context kept) | Esc |
| Restore a previous conversation/code state | Esc + Esc or `/rewind` |
| Revert the last changes | "Undo that" |
| Reset the context between unrelated tasks | `/clear` |
| Compact with a focus | `/compact Focus on the API changes` |
| Ask a side question that never enters history | `/btw` |
| Name the session so you can find it later | `/rename oauth-migration` |
| Continue the most recent session | `claude --continue` |
| Choose a session from a list | `claude --resume` |

> **Checkpoint limit:** checkpoints track only changes Claude makes with its file-edit tools. Changes from Bash commands or external processes are not captured. A checkpoint is not a replacement for git.

> **Two-correction rule:** if you've corrected Claude more than twice on the same problem, run `/clear` and write a better first prompt that includes what you learned.

### 8. Automate and scale

**Non-interactive mode:**

```bash
# One-off query — plain text
claude -p "Explain what this project does"

# Structured output for scripts — one JSON object with a result field
claude -p "List all API endpoints" --output-format json

# Streaming — one JSON object per line, starting with an init event
claude -p "Analyse this log file" --output-format stream-json --verbose

# Unattended run with background safety checks
claude --permission-mode auto -p "fix all lint errors"
```

**Fan out across files:**

```bash
# 1. Ask Claude to write the task list to a file
# 2. Loop over the list
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
# 3. Test on 2-3 files, refine the prompt, then run the full set
```

| Parallel option | What you get |
| --- | --- |
| Worktrees | Separate CLI sessions in isolated git checkouts, so edits do not collide |
| Desktop app | Several local sessions managed visually, each in its own worktree |
| Claude Code on the web | Sessions in the cloud, on Anthropic-managed infrastructure by default |
| Agent teams | Automatic coordination of several sessions with shared tasks, messaging, and a team lead |

### 9. Add an adversarial review step

A reviewer in a fresh subagent context sees only the diff and your criteria, not the reasoning that produced the change.

- For a correctness check, run the bundled `/code-review` skill.
- To check the diff against your plan, write the review prompt yourself:

```text
Use a subagent to review the rate limiter diff against PLAN.md. Check that
every requirement is implemented, the listed edge cases have tests, and
nothing outside the task's scope changed. Report gaps, not style preferences.
```

> Do not chase every finding. Tell the reviewer to report only gaps that affect correctness or a stated requirement. Treat the rest as optional.

### 10. Develop your own judgement

These patterns are starting points, not rules. Sometimes you must let context accumulate, skip the plan, or use a vague prompt on purpose. Watch what works: when output is good, note the prompt structure, context, and mode. When Claude struggles, ask why — noisy context? vague prompt? task too large for one pass?

---
## Multi-Agent Design Patterns

### Pattern 1: Three-Stage Pipeline

```
PM-Spec Agent → Architect-Review Agent → Implementer-Tester Agent
     │                   │                        │
     ▼                   ▼                        ▼
READY_FOR_ARCH     READY_FOR_BUILD              DONE
```

Status flow: `BACKLOG → READY_FOR_ARCH → READY_FOR_BUILD → DONE`, tracked in a queue file. Handoffs can be automated with SubagentStop/Stop hooks.

### Pattern 2: Writer/Reviewer Separation

| Session A (Writer) | Session B (Reviewer) |
| --- | --- |
| Implement the feature | |
| | Review the implementation. Focus on edge cases, race conditions, consistency. |
| Address review feedback | |

A fresh context improves review quality — Claude won't be biased toward code it just wrote.

### Pattern 3: Competing Hypotheses (Agent Teams)

Multiple teammates test different theories simultaneously and challenge each other. Adversarial parallel investigation finds the strongest surviving theory.

### Pattern 4: Domain-Split Parallel Work

```
Team:
├── Frontend teammate → src/components/
├── Backend teammate  → src/api/
├── Test teammate     → tests/ (depends on both above)
└── Docs teammate     → docs/
```

Task dependencies ensure proper ordering while maximising parallelism.

### Pattern 5: Chained Subagents

Sequential subagent invocations from the main conversation — each completes and returns results, passed as context to the next.

### Pattern 6: MCP Bridge to External LLM

Build a lightweight MCP server that calls an external model for quality checks at pipeline stages.

### Pattern 7: Research Fan-Out (Subagents)

Spawn multiple subagents simultaneously for independent investigations. Claude synthesises findings.

> **Warning:** When subagents complete, results return to main conversation. Many detailed results consume significant context.

---

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Fix |
| --- | --- | --- |
| Same-File Editing | Two agents editing the same file leads to overwrites | Ensure file ownership is disjoint |
| Excessive Parallelism | More agents ≠ better. Token costs scale linearly; coordination scales worse | 3 focused agents outperform 5 scattered ones |
| Missing HITL Gates | Agents make irreversible decisions without approval | Pause for human approval at critical points |
| Implicit Communication | Relying on agents inferring what other agents did | Use explicit mechanisms: hooks, task lists, direct messaging |
| Unscoped Investigation | Agent told to "investigate" reads hundreds of files, filling context | Always scope investigations narrowly |

### Common Failure Patterns (Sessions)

| Anti-Pattern | Symptom | Fix |
| --- | --- | --- |
| Kitchen sink session | Start with one task, ask unrelated things, go back | `/clear` between unrelated tasks |
| Correcting repeatedly | Wrong → correct → still wrong → correct again | After 2 failures, `/clear` and write a better initial prompt |
| Over-specified CLAUDE.md | Claude ignores rules because important ones lost in noise | Ruthlessly prune; convert working defaults to hooks |
| Trust-then-verify gap | Plausible implementation that doesn't handle edge cases | Always provide verification (tests, scripts, screenshots) |
| Infinite exploration | Unscoped investigation reads hundreds of files | Scope narrowly or use subagents |

---

## Permission Distribution Guidelines

| Agent Role | Recommended Tools |
| --- | --- |
| PM / Architect | Read-heavy: Read, Grep, Glob, WebSearch, MCP docs |
| Implementer | Full: Read, Edit, Write, Bash, MCP (Playwright) |
| Reviewer | Read-only: Read, Grep, Glob, Bash (limited) |
| Release | Minimal: only essential deployment tools |

---
## Context Management Strategies

### What Consumes Context

| Source | Context Cost | Persistence |
| --- | --- | --- |
| Your messages | Proportional to length | Until compaction |
| File reads (Read) | Full file contents | Until compaction |
| Command outputs (Bash) | Full stdout/stderr | Until compaction |
| CLAUDE.md | Full content | Every request |
| Auto memory | First 200 lines | Every request |
| MCP tool definitions | All schemas | Every request |
| Skill descriptions | Names + descriptions (small) | Every request |
| Subagent results | Summary only | Until compaction |
| Hooks | Zero | External execution |

### Strategy 1: Proactive Context Hygiene

- `/clear` — Reset between unrelated tasks
- `/compact Focus on the API changes` — Targeted compaction
- `/rewind` → Summarise from here — Partial compaction
- `/context` — Check what's using space
- `/btw` — Side questions that don't enter history

### Strategy 2: Delegate to Subagents

Subagents run in separate context windows. High-volume operations (test suites, codebase exploration, post-implementation verification) stay in the subagent's context — only a summary returns.

### Strategy 3: Minimise Always-On Context

| Feature | Optimisation |
| --- | --- |
| CLAUDE.md | Keep under 200 lines. Move reference material to skills |
| MCP servers | Disconnect unused servers. Run `/mcp` to check costs |
| Skills | Use `disable-model-invocation: true` for manual-only skills |

### Strategy 4: Compaction Configuration

```text
When compacting, always preserve:
- The full list of modified files
- Any test commands and their results
- Architecture decisions made in this session
```

Auto-compaction triggers at ~95% capacity. Override with `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50` for earlier compaction.

### Context Budget Mental Model

```
┌─────────────────────────────────────────────────┐
│                 Context Window                   │
├─────────────────────────────────────────────────┤
│ Fixed overhead (every request):                  │
│   ├── System prompt                              │
│   ├── CLAUDE.md                                  │
│   ├── Auto memory (200 lines)                    │
│   ├── MCP tool definitions                       │
│   └── Skill descriptions                         │
├─────────────────────────────────────────────────┤
│ Variable (grows with session):                   │
│   ├── Conversation history                       │
│   ├── File contents from Read                    │
│   ├── Command outputs from Bash                  │
│   ├── Loaded skill full content                  │
│   └── Subagent result summaries                  │
├─────────────────────────────────────────────────┤
│ Isolated (does NOT consume main context):        │
│   ├── Subagent internal work                     │
│   ├── Agent team teammate contexts               │
│   └── Hook execution                             │
└─────────────────────────────────────────────────┘
```