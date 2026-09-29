# 04_L01_Agentic loop

# Lab 1 — Agentic Loop & Context

> Debug intermittent 500 errors using the **Explore → Plan → Code** workflow.

---

## Scenario

A legacy Node.js API has intermittent 500 errors. The stack traces point vaguely to the auth middleware. You do not know the codebase layout, and a naive "fix it" prompt will pull in 40 files Claude does not need.

> **Goal:** Practice the Explore → Plan → Code workflow to investigate a production bug without burning your context window on irrelevant files.

### Setup Assumptions

- You are in a local clone of the repo with read/write access.
- No production access — use test fixtures and local logs only.
- Tests exist and can be run with `npm test`.
- Do not ask Claude to open every file at once; context is a finite budget.

---

## Exercise Steps

### Step 1 — Fire an Explore Subagent

Map the blast radius without modifying anything:

```text
Use the Explore subagent to map the auth middleware and its dependencies.
Do not edit files. Return: entry point, middleware chain order, files that
touch session or JWT, and the last test that exercises this path.
```

### Step 2 — Read Only Flagged Files

Read only the files the Explore result flagged:

```text
Read src/middleware/auth.js, src/services/session.js, and
test/auth.test.js. Identify where a thrown error could become an
unhandled 500 rather than a 401.
```

### Step 3 — Ask for a Scoped Plan

Get a plan before any edits:

```text
Propose a fix plan. Include: exact lines to change, which test to update,
and the npm test command to verify. Do not edit yet. One sentence on risk.
```

### Step 4 — Implement the Minimal Change

Approve and implement:

```text
Implement only the error-handling fix you proposed. Touch no other files.
Run npm test after each edit and report pass/fail inline.
```

### Step 5 — Verify and Close

Summarise the work done:

```text
Summarise: files changed, lines changed, tests updated, test output,
and any residual risk. Then /clear is safe.
```

---

## Expected Claude Code Behaviour

Claude opens minimal files, stages a plan before touching code, runs tests inline, and ends with a summary tight enough that `/clear` loses nothing important.

### Success Criteria

- First response names files without editing them.
- Plan step lists exact lines and the verification command.
- Implementation produces a diff under 30 lines.
- `npm test` passes before Claude summarises.
- Summary is self-contained — no follow-up questions needed after `/clear`.

> **Reflection:** At which point in the workflow would `/clear` be safe to call without losing critical context, and what would you move into CLAUDE.md to avoid reloading that context next time?