# 16_Headless & CI-CD

# M11 — Headless Mode & CI/CD

> `claude -p`, `--bare` mode, output formats, JSON schema, GitHub Actions, and GitLab CI integration.

---

## What Is Headless Mode?

Headless mode (`claude -p`) runs Claude Code non-interactively — no session, no terminal UI. Use it in CI pipelines, pre-commit hooks, scripts, and automation workflows.

The Agent SDK provides the same tools, agent loop, and context management that power Claude Code. Available as CLI (`claude -p`), Python, and TypeScript packages.

---

## Basic Usage

```bash
# Simple query
claude -p "What does the auth module do?"

# With tool restrictions
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"

# Structured JSON output
claude -p "Summarize this project" --output-format json

# Streaming JSON
claude -p "Analyse this log file" --output-format stream-json
```

---

## Bare Mode (`--bare`)

Skips auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and CLAUDE.md. Only flags you pass explicitly take effect.

```bash
claude --bare -p "Summarize this file" --allowedTools "Read"
```

> `--bare` will become the default for `-p` in a future release.

**Loading context in bare mode:**

| To Load | Flag |
| --- | --- |
| System prompt additions | `--append-system-prompt`, `--append-system-prompt-file` |
| Settings | `--settings <file-or-json>` |
| MCP servers | `--mcp-config <file-or-json>` |
| Custom agents | `--agents <json>` |
| Plugin directory | `--plugin-dir <path>` |

> **Auth note:** Bare mode skips OAuth/keychain. Use `ANTHROPIC_API_KEY` env var or `apiKeyHelper` in `--settings`.

---

## Output Formats

| Format | Flag | Use Case |
| --- | --- | --- |
| Plain text | (default) | Human-readable output |
| JSON | `--output-format json` | Structured parsing with session metadata |
| Stream JSON | `--output-format stream-json` | Real-time streaming, token-by-token |

**JSON schema (structured output):**

```bash
claude -p "Extract main function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}'
```

Returns structured data in the `structured_output` field.

**Streaming with token deltas:**

```bash
claude -p "Write a poem" --output-format stream-json --verbose --include-partial-messages | \
  jq -rj 'select(.type == "stream_event" and .event.delta.type? == "text_delta") | .event.delta.text'
```

---

## Auto-Approve Tools

```bash
claude -p "Run the test suite and fix any failures" \
  --allowedTools "Bash,Read,Edit"

# Allow specific git commands (space before * is important)
--allowedTools "Bash(git diff *),Bash(git log *),Bash(git status *),Bash(git commit *)"
```

> **Watch the space:** `Bash(git diff*)` without a space would also match `git diff-index`. Use `Bash(git diff *)` for exact prefix matching.

---

## Continuing Conversations

```bash
# First request
claude -p "Review this codebase for performance issues"

# Continue most recent conversation
claude -p "Now focus on the database queries" --continue

# Capture session ID for specific continuation
session_id=$(claude -p "Start a review" --output-format json | jq -r '.session_id')
claude -p "Continue that review" --resume "$session_id"
```

---

## System Prompt Customisation

```bash
# Append to default system prompt
gh pr diff "$1" | claude -p \
  --append-system-prompt "You are a security engineer. Review for vulnerabilities." \
  --output-format json

# Fully replace system prompt
claude -p "Analyse code" --system-prompt "You are a code auditor..."
```

---

## Practical CI/CD Patterns

**Create a commit from staged changes:**

```bash
claude -p "Look at my staged changes and create an appropriate commit" \
  --allowedTools "Bash(git diff *),Bash(git log *),Bash(git status *),Bash(git commit *)"
```

**Fan out across files:**

```bash
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
```

---

## GitHub Actions Integration

**Respond to @claude mentions:**

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
jobs:
  claude:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

**PR review on every push:**

```yaml
name: Code Review
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "Review this pull request for code quality, correctness, and security."
          claude_args: "--max-turns 5"
```

**Scheduled automation** — use an `on: schedule: cron` trigger with a `prompt` (e.g. "Generate a summary of yesterday's commits and open issues") and `claude_args: "--model opus"`.

### Action Parameters

| Parameter | Description | Required |
| --- | --- | --- |
| `prompt` | Instructions for Claude | No* |
| `claude_args` | CLI arguments | No |
| `anthropic_api_key` | API key | Yes** |
| `github_token` | GitHub token | No |
| `trigger_phrase` | Custom trigger (default: `@claude`) | No |
| `use_bedrock` | Use AWS Bedrock | No |
| `use_vertex` | Use Google Vertex AI | No |

\*Optional — when omitted, Claude responds to trigger phrase mentions. \*\*Required for direct API; not for Bedrock/Vertex.

### Cloud Provider Support

| Provider | Authentication |
| --- | --- |
| AWS Bedrock | OIDC + IAM role (`aws-actions/configure-aws-credentials`) |
| Google Vertex AI | Workload Identity Federation (`google-github-actions/auth`) |
| Direct API | `ANTHROPIC_API_KEY` secret |

---

## GitLab CI/CD

Same Agent SDK, different platform. Use `claude -p` with `--bare` in `.gitlab-ci.yml`:

```yaml
code-review:
  script:
    - claude --bare -p "Review the MR changes" --allowedTools "Read,Grep,Glob" --output-format json
```

---

## API Retry Events (Streaming)

When streaming, failed API requests emit `system/api_retry` events:

| Field | Description |
| --- | --- |
| `attempt` | Current attempt number (starting at 1) |
| `max_retries` | Total retries permitted |
| `retry_delay_ms` | Milliseconds until next attempt |
| `error_status` | HTTP status code or `null` |
| `error` | Error category |