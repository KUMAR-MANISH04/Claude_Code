# 09_L03_Subagents

# Lab 3 — Subagents

> Dispatch 3 parallel review subagents for a PR.

---

## Scenario

A PR modifies JWT token rotation (auth), adds a Stripe webhook handler (payments), and rewrites the checkout form (UI). One reviewer cannot hold all three lenses clearly. You will use subagents to run them simultaneously and synthesise a single verdict.

> **Goal:** Dispatch three independent review subagents in parallel for a PR that touches auth, payments, and UI — without polluting the main context.

### Setup Assumptions

- You have the PR diff available locally or via a git range.
- Each subagent receives only its relevant files, not the full diff.
- Subagents return summaries, not full file contents.
- Main session is Sonnet; subagents can use Haiku for read-only review tasks.

---

## Exercise Steps

### Step 1 — Security Review Subagent

```text
Dispatch a subagent to review only the auth changes in this diff:
src/auth/token.js, src/middleware/jwt.js.
Check for: token expiry enforcement, rotation race conditions, secret
leakage in logs. Return a 5-bullet summary with severity (LOW/MED/HIGH).
The subagent must not read payment or UI files.
```

### Step 2 — Test Coverage Subagent (Parallel)

```text
Dispatch a subagent to review test coverage for the payments changes:
src/webhooks/stripe.js, test/webhooks/stripe.test.js.
Check for: unhappy path coverage, idempotency key tests, payload
validation tests. Return a gap list ordered by risk. Max 200 words.
```

### Step 3 — UX Regression Subagent (Parallel)

```text
Dispatch a subagent to review the checkout form changes in
src/components/CheckoutForm.tsx.
Check for: accessibility regressions (aria labels, tab order), loading
state handling, error message clarity. Return findings as a checklist.
```

### Step 4 — Synthesise All Reports

```text
Synthesise the three subagent reports into a single PR verdict.
Structure: APPROVE / REQUEST CHANGES / BLOCK, followed by top 3 issues
by risk, followed by optional improvements. Max 300 words.
```

---

## Expected Claude Code Behaviour

Subagents run without leaking context between lanes. Each returns a bounded summary. The main session synthesises without re-reading source files. Total token cost is lower than a single-agent full-diff review.

### Success Criteria

- Each subagent reads only its assigned files.
- All three subagent summaries are returned before synthesis begins.
- The final verdict names severity and links each finding to a file.
- Main session context does not contain raw source code from the PR.
- You can articulate why this approach costs less than a single-agent review.

> **Reflection:** Subagents add orchestration overhead (~7× per agent call). At what PR complexity does parallel subagent review become cheaper than a single thorough pass, and when would you stay in-session instead?