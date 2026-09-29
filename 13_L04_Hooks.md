# 13_L04_Hooks

# Lab 4 — Hooks

> Write a PreToolUse hook blocking `.env` edits and a Stop hook enforcing test runs.

---

## Scenario

Your team keeps accidentally editing `.env` files and pushing without running the test suite. Both failures happened in the last sprint. Hooks enforce these guardrails at the tool level, not the prompt level.

> **Goal:** Write and test two hooks: a PreToolUse hook that blocks `.env` edits, and a Stop hook that runs tests before Claude exits.

### Setup Assumptions

- You have write access to `.claude/settings.json` in the repo root.
- A test command exists: `npm test` (or equivalent for your stack).
- Claude Code 1.x — hooks execute shell commands and receive JSON on stdin.
- You will test each hook by deliberately triggering the blocked behaviour.

---

## Exercise Steps

### Step 1 — Write the PreToolUse Hook Script

```text
Write a bash script at .claude/hooks/block-env-edits.sh that reads
a PreToolUse JSON payload from stdin, checks if the tool is Write or Edit
and the file path ends in .env, and exits 2 with a clear error message if
so. Exit 0 for all other cases.
```

### Step 2 — Write the Stop Hook Script

```text
Write a bash script at .claude/hooks/run-tests-on-stop.sh that runs
npm test and exits with the test process exit code. If tests fail, exit 2
with a message: "Tests failed — Claude cannot stop until tests pass."
```

### Step 3 — Wire Hooks into Settings

```text
Add both hooks to .claude/settings.json. The PreToolUse hook should
match on tool names Write and Edit. The Stop hook should run on the
Stop event. Show the complete hooks section of the JSON.
```

### Step 4 — Test the `.env` Block

```text
Try to add a comment line to .env. Report what Claude Code does —
does it proceed or does the hook stop it?
```

### Step 5 — Test the Stop Hook

```text
Introduce a deliberate syntax error in a test file, then attempt to
finish the session. Report the hook output and exit behaviour.
```

---

## Expected Claude Code Behaviour

The PreToolUse hook intercepts Write/Edit calls targeting `.env` files and returns exit code 2, causing Claude to abort and surface the error message. The Stop hook blocks session end when tests fail, forcing a fix cycle before Claude can stop.

> **Exit Codes:** Exit code `2` = block and surface error. Exit code `0` = allow. Exit code `1` = hook failure (silent pass-through).

### Success Criteria

- `block-env-edits.sh` exits 2 when a `.env` path is detected, 0 otherwise.
- `run-tests-on-stop.sh` exits with the test runner's exit code.
- `.claude/settings.json` has a valid hooks section for both events.
- Attempting to edit `.env` produces the hook's error message, not a file change.
- A failing test prevents Claude from stopping and surfaces the failure output.

> **Reflection:** Exit code 2 blocks and surfaces an error; exit code 0 allows. What would exit code 1 do, and what is the security implication of putting hook scripts in the repo versus user-level settings?