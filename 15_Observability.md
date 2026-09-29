# 15_Observability

# M10 — Observability (OpenTelemetry)

> OTel quick start, 8 metrics, 4 event types, standard attributes, configuration, and dashboard recommendations.

---

## Overview

Claude Code exports telemetry via OpenTelemetry (OTel) — metrics as time series via the standard metrics protocol, and events via the logs/events protocol.

> **Telemetry is opt-in** and requires explicit configuration.

---

## Quick Start

```bash
# 1. Enable telemetry
export CLAUDE_CODE_ENABLE_TELEMETRY=1

# 2. Choose exporters
export OTEL_METRICS_EXPORTER=otlp          # otlp | prometheus | console
export OTEL_LOGS_EXPORTER=otlp             # otlp | console

# 3. Configure endpoint
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# 4. Authentication (if required)
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer your-token"

# 5. Run Claude Code
claude
```

> Default intervals: 60s for metrics, 5s for logs.

---

## All 8 Metrics

| Metric | Unit | Description |
| --- | --- | --- |
| `claude_code.session.count` | count | CLI sessions started |
| `claude_code.lines_of_code.count` | count | Lines of code modified |
| `claude_code.pull_request.count` | count | Pull requests created |
| `claude_code.commit.count` | count | Git commits created |
| `claude_code.cost.usage` | USD | Cost of session |
| `claude_code.token.usage` | tokens | Tokens used (input/output/cacheRead/cacheCreation) |
| `claude_code.code_edit_tool.decision` | count | Edit tool permission decisions (accept/reject) |
| `claude_code.active_time.total` | seconds | Active time (user keyboard + CLI processing) |

- **Token usage** attributes: `type` (`input`, `output`, `cacheRead`, `cacheCreation`) and `model`.
- **Active time** attributes: `type` is `user` (keyboard) or `cli` (tool execution + AI responses).

---

## All 4 Event Types

Events are exported via OTel logs (requires `OTEL_LOGS_EXPORTER`).

**1. User Prompt (`claude_code.user_prompt`)** — `prompt_length`; `prompt` content (redacted by default, enable with `OTEL_LOG_USER_PROMPTS=1`).

**2. Tool Result (`claude_code.tool_result`)** — `tool_name`, `success`, `duration_ms`, `error`, `decision_type` (accept/reject), `decision_source` (config/hook/user_*), `tool_result_size_bytes`, `tool_parameters`.

**3. API Request (`claude_code.api_request`)** — `model`, `cost_usd`, `duration_ms`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_creation_tokens`, `speed` (fast/normal).

**4. API Error (`claude_code.api_error`)** — `error`, `status_code`, `attempt`.

> **Event correlation:** All events share a `prompt.id` (UUID v4) linking them to the triggering user prompt. Filter by `prompt.id` to trace all activity from a single prompt.

---

## Standard Attributes (All Metrics & Events)

| Attribute | Description | Default |
| --- | --- | --- |
| `session.id` | Unique session identifier | Included (disable: `OTEL_METRICS_INCLUDE_SESSION_ID=false`) |
| `app.version` | Claude Code version | Not included (enable: `OTEL_METRICS_INCLUDE_VERSION=true`) |
| `organization.id` | Org UUID | Always when available |
| `user.account_uuid` | Account UUID | Included (disable: `OTEL_METRICS_INCLUDE_ACCOUNT_UUID=false`) |
| `user.id` | Anonymous device/installation ID | Always |
| `user.email` | User email (OAuth only) | Always when available |
| `terminal.type` | Terminal type (iTerm, vscode, tmux) | Always when detected |

---

## Configuration Reference

### Core Variables

| Variable | Description | Values |
| --- | --- | --- |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | Enable telemetry (required) | `1` |
| `OTEL_METRICS_EXPORTER` | Metrics exporters | `console`, `otlp`, `prometheus` |
| `OTEL_LOGS_EXPORTER` | Logs/events exporters | `console`, `otlp` |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | OTLP protocol | `grpc`, `http/json`, `http/protobuf` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint | `http://localhost:4317` |
| `OTEL_METRIC_EXPORT_INTERVAL` | Metrics interval (ms) | Default: `60000` |
| `OTEL_LOGS_EXPORT_INTERVAL` | Logs interval (ms) | Default: `5000` |

### Privacy Controls

| Variable | Description | Default |
| --- | --- | --- |
| `OTEL_LOG_USER_PROMPTS` | Include prompt content | Disabled |
| `OTEL_LOG_TOOL_DETAILS` | Include MCP server/tool names, skill names | Disabled |

**Multi-team segmentation:**

```bash
export OTEL_RESOURCE_ATTRIBUTES="department=engineering,team.id=platform,cost_center=eng-123"
```

**Admin configuration (managed settings)** — set the same env vars in managed `settings.json`; managed settings have high precedence and cannot be overridden by users.

---

## What to Monitor

**Usage dashboards**

| Metric | Analysis |
| --- | --- |
| `token.usage` by type | Input/output/cache breakdown per user/team/model |
| `session.count` | Adoption and engagement trends |
| `lines_of_code.count` | Productivity (additions vs removals) |
| `commit.count` + `pull_request.count` | Development workflow impact |

**Cost alerts** — cost spikes per user/team, unusual token consumption, high session volume from specific accounts.

**Tool usage analysis** — most-used tools, success rates, average execution times, error patterns by tool type.

**Backend recommendations**

| Backend | Best For |
| --- | --- |
| Prometheus | Rate calculations, aggregated metrics |
| ClickHouse | Complex queries, unique user analysis |
| Datadog / Honeycomb | Advanced querying, visualisation, alerting |
| Elasticsearch / Loki | Full-text search, log analysis |