# 18_ MCP Integration

# M13 — MCP Integration

> Adding MCP servers, scope hierarchy, 4 transport types, tool search, MCP in subagents, security, elicitation, and resources.

---

## What Is MCP?

Model Context Protocol (MCP) connects Claude Code to external tools and data sources — databases, APIs, browsers, file systems, monitoring, and more. MCP servers expose tools that Claude can call during its agentic loop.

---

## Adding MCP Servers

**Interactive CLI:**

```bash
# stdio transport (local binary)
claude mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem /workspace

# HTTP transport (remote server)
claude mcp add stripe --transport http --url https://mcp.stripe.com \
  --header "Authorization: Bearer $STRIPE_MCP_KEY"
```

**Configuration file (`.mcp.json`)** — place at project root for project-scoped servers:

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    },
    "stripe": {
      "type": "http",
      "url": "https://mcp.stripe.com",
      "headers": {
        "Authorization": "Bearer ${STRIPE_MCP_KEY}"
      }
    },
    "analytics": {
      "type": "sse",
      "url": "http://localhost:3100/sse"
    }
  }
}
```

---

## Scope Hierarchy

| Location | Scope | Priority |
| --- | --- | --- |
| `.mcp.json` (project root) | Project | 1 (highest — "local") |
| `.claude/settings.json` | Project | 2 |
| `~/.claude/settings.json` | All projects | 3 (lowest — "user") |

> Override by name: higher priority wins.

---

## 4 Transport Types

| Transport | When to Use | Configuration |
| --- | --- | --- |
| stdio | Local binaries, npm packages | `command` + `args` |
| http | Remote MCP servers | `url` + optional `headers` |
| sse | Server-Sent Events streaming | `url` |
| ws | WebSocket connections | `url` |

> **Environment variable interpolation:** Use `${VAR_NAME}` in configuration — resolved at runtime.

---

## Tool Search (Automatic Scaling)

When MCP tool descriptions exceed 10% of your context window, Claude Code automatically defers them and loads tools on-demand.

1. Tool descriptions above threshold are NOT loaded into context.
2. Claude uses `ToolSearch` to find relevant tools when needed.
3. Only matched tools enter context.
4. Reduces idle tool definition overhead.

```bash
ENABLE_TOOL_SEARCH=auto:5    # Trigger at 5% context instead of 10%
```

### CLI vs MCP: Cost Comparison

| Approach | Context Cost | When to Prefer |
| --- | --- | --- |
| CLI tools (`gh`, `aws`, `gcloud`) | Zero (per-command only) | Tool available as CLI |
| MCP servers | Every request (tool definitions) | Need structured tool interface, complex interactions |

> **Rule of thumb:** If a CLI tool exists, prefer it over an MCP server. Claude can run CLI commands directly without persistent context overhead.

---

## MCP in Subagents

Define MCP servers inline in subagent frontmatter to scope them:

```yaml
---
name: browser-tester
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github    # Reference existing server by name
---
```

> **Key benefit:** Inline MCP servers keep tool definitions **out of the main conversation context**. The subagent gets the tools; the parent does not.

---

## MCP Tool Naming Convention

MCP tools in hooks, permissions, and logs use the pattern `mcp__<server>__<tool>`:

```
mcp__github__search_repositories
mcp__filesystem__read_file
mcp__memory__create_entities
```

Use in hook matchers and permission rules:

```json
{
  "matcher": "mcp__github__.*",
  "permissions": { "allow": ["mcp__filesystem__read_file"] }
}
```

---

## MCP Security

**Authentication:**

```json
{
  "headers": {
    "Authorization": "Bearer ${API_KEY}",
    "X-API-Key": "${ANOTHER_KEY}"
  }
}
```

**Permission scoping:**

- Only expose tools Claude needs (read-only if writes aren't required).
- Use `permissions.deny` to block specific MCP tools.
- Use sandbox `allowedDomains` to restrict which domains MCP servers can reach.

**Middleware validation via hooks:**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__filesystem__write_file",
        "hooks": [
          { "type": "command", "command": "./scripts/validate-mcp-write.sh" }
        ]
      }
    ]
  }
}
```

> **Rate limiting:** MCP tools can have side effects. Use hooks to implement rate limiting on tool calls that modify external state.

---

## MCP Elicitation & Resources

**Elicitation** — MCP servers can request user input mid-task. Claude Code shows a form or URL dialog. Hook events for programmatic handling:

- `Elicitation` — intercept before showing to user.
- `ElicitationResult` — modify response before sending to server.

**Resources** — MCP servers can expose read-only data:

| Tool | Description |
| --- | --- |
| `ListMcpResourcesTool` | List available resources from connected servers |
| `ReadMcpResourceTool` | Read a specific resource by URI |

---

## Practical Setup Checklist

1. Install MCP server: `claude mcp add <name> -- <command>`
2. Verify connection: `/mcp`
3. Check context cost: `/mcp` → review token usage per server
4. Disconnect unused servers to save context
5. Use tool search if many tools: `ENABLE_TOOL_SEARCH=auto:5`
6. Scope MCP to subagents when possible to keep main context clean
7. Add authentication for HTTP/remote servers
8. Use hook validation for MCP tools with side effects