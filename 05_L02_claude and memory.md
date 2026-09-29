# 05_L02_claude and memory

# Lab 2 — CLAUDE.md & Memory

> Configure CLAUDE.md and auto-memory for a Python monorepo.

---

## Scenario

A new team member joins a repo with 3 services (`api`, `worker`, `shared`). Every session they spend the first 10 minutes explaining the stack, the run commands, and the no-direct-DB-writes rule. You will fix this permanently.

> **Goal:** Configure CLAUDE.md and auto-memory so Claude Code understands a Python monorepo without re-explanation every session.

### Setup Assumptions

- Python monorepo with directories: `api/`, `worker/`, `shared/`, root `Makefile`.
- Each service has its own `pyproject.toml` and test suite.
- You have write access to the repo root.
- Claude Code is version 1.x with auto-memory enabled.

---

## Exercise Steps

### Step 1 — Draft a Root CLAUDE.md

Cover the four mandatory sections:

```text
Write a CLAUDE.md for this monorepo. Include: project summary (2 sentences),
commands (how to run each service, how to run all tests, how to lint),
conventions (import order, service boundary rules, no direct DB writes from
worker), and safety rules (never edit migration files, always run mypy before
committing). Keep it under 500 lines.
```

### Step 2 — Verify Claude Reads It Automatically

Confirm CLAUDE.md is picked up without prompting:

```text
What are the run commands for each service and what is the safety rule
about database writes? Answer from CLAUDE.md only — do not search files.
```

### Step 3 — Trigger Auto-Memory

Surface a new architectural insight:

```text
I just discovered that shared/events.py is the canonical schema registry.
Any new event type must be added there first. Remember this.
```

### Step 4 — Confirm Persistence Survives `/clear`

Test that memory persists across context resets:

```text
/clear
What is the canonical schema registry file and why does it matter?
```

### Step 5 — Inspect the Memory Entry

Verify where the memory was saved:

```text
Show me the auto-memory entry that was created for the schema registry
discovery. What file is it in and what does it say?
```

---

## Expected Claude Code Behaviour

Claude reads CLAUDE.md automatically at session start, enforces safety rules without being reminded, and persists the schema registry note to the project memory file. After `/clear` the insight survives because it lives in memory, not the conversation.

### Success Criteria

- CLAUDE.md is under 500 lines and covers all four sections.
- Claude correctly answers the run-commands question from CLAUDE.md alone.
- Auto-memory creates a new entry for the schema registry insight.
- After `/clear`, Claude recalls the schema registry without re-explanation.
- You can identify which file holds the persisted memory entry.

> **Reflection:** CLAUDE.md, auto-memory, and memory files serve different purposes — what is each one optimised for, and what happens if you put team conventions in auto-memory instead of CLAUDE.md?