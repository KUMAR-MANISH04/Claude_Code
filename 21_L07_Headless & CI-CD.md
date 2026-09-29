# 21_L07_Headless & CI-CD

# Lab 7 — Headless & CI/CD

> Wire Claude Code into a GitHub Actions PR review.

---

## Scenario

Your team reviews every PR manually. You want Claude to post an automated review comment within 60 seconds of a PR being opened, covering diff quality, test coverage gaps, and obvious bugs — with no human in the loop.

> **Goal:** Run Claude Code non-interactively via `claude -p`, wire it into a GitHub Actions PR review workflow, and verify deterministic CI behaviour.

### Setup Assumptions

- You have a GitHub repo with Actions enabled.
- `ANTHROPIC_API_KEY` is stored as a GitHub Actions secret.
- Claude Code CLI is installable in CI via npm.
- The PR diff is accessible via `gh pr diff`.

---

## Exercise Steps

### Step 1 — Test Headless Command Locally

```bash
gh pr diff | claude -p "Review this diff. Flag bugs, missing tests,
and style issues. Be concise." --output-format json

# What fields does the JSON envelope contain?
```

### Step 2 — Test the `--bare` Flag

```text
Run the same prompt with --bare instead of --output-format json.
Show the difference in output format. When would you prefer --bare
over JSON in a pipeline?
```

### Step 3 — Write the GitHub Actions Workflow

```text
Write a GitHub Actions workflow at .github/workflows/claude-review.yml
that triggers on pull_request (opened, synchronize), installs claude CLI,
runs a diff review with --output-format json, extracts the text field,
and posts it as a PR comment using gh pr comment. Use
${{ secrets.ANTHROPIC_API_KEY }} for auth. Set allowed tools to
Bash and Read only.
```

### Step 4 — Add Determinism Guards

```text
Add to the workflow: CLAUDE_NO_USER_HOOKS=1 environment variable,
a fixed model pin (do not use "latest"), and a --max-turns limit of 5.
Explain why each guard matters for reproducible CI output.
```

### Step 5 — Verify the Workflow

```text
Describe the exact sequence of steps Claude Code executes in this
workflow from Action trigger to PR comment posted. Identify the step
most likely to fail on a cold CI runner and how to fix it.
```

---

## Expected Claude Code Behaviour

`claude -p` exits after producing output without waiting for user input. The JSON envelope is parseable. The workflow file is valid YAML with correct secret references. Determinism guards are present and explained.

> **CI Safety:** Always set `CLAUDE_NO_USER_HOOKS=1` in CI to prevent local hook scripts from affecting automated runs. Pin the model version to avoid surprise behaviour changes.

### Success Criteria

- Local `claude -p` run produces valid JSON with a text field.
- `--bare` output is plain text with no envelope — you can explain when to use each.
- Workflow YAML is valid, triggers on PR events, and posts a comment.
- `CLAUDE_NO_USER_HOOKS=1`, model pin, and `--max-turns` are all present.
- You can identify the most fragile CI step and its mitigation.

> **Reflection:** `claude -p` is deterministic by design — same prompt, same diff, consistent output. What inputs still introduce variability in CI reviews, and how would you measure review quality over time?