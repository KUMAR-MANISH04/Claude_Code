# 22_L08_MCP & Plugins

# Lab 8 — MCP & Plugins

> Connect 3 MCP servers, measure overhead, and scope to subagents.

---

## Scenario

Your team uses GitHub for issues and PRs, a Postgres database for query analysis, and Playwright for e2e test automation. You want Claude to access all three without writing custom integration code — and without every MCP server inflating every session's context.

> **Goal:** Connect three MCP servers (GitHub, Postgres, Playwright), measure their context overhead, scope one server to a subagent only, and explore the plugin marketplace.

### Setup Assumptions

- Claude Code 1.x with MCP support enabled.
- You have a local Postgres instance or a safe dev database.
- GitHub personal access token available as an environment variable.
- Playwright is installed in the repo (`npm install @playwright/test`).
- No production systems — dev/staging credentials only.

---

## Exercise Steps

### Step 1 — Install and Verify MCP Servers

```text
Add three MCP servers via /mcp add:
- github (name: github, command: npx @modelcontextprotocol/server-github)
- postgres (name: postgres, command: npx @modelcontextprotocol/server-postgres,
  env: DATABASE_URL=postgresql://localhost:5432/dev)
- playwright (name: playwright,
  command: npx @modelcontextprotocol/server-playwright)

After adding each, list the tools it registers. How many tools total?
```

### Step 2 — Measure Context Overhead

```text
Run /context before and after adding all three MCP servers (use /clear
between readings). Report the token delta. Which server added the most
tool-registration tokens? Is the overhead per-session or per-call?
```

### Step 3 — Use the GitHub MCP Server

```text
Use the github MCP tool to fetch the last 3 open pull requests in this
repo. For each, return: PR number, title, author, and files changed count.
Do not read the full diff — use only the PR metadata tools.
```

### Step 4 — Scope Playwright to Subagent Only

```text
Remove the playwright MCP server from the main session scope. Then
dispatch a subagent with playwright scoped to it, and have the subagent
run a Playwright screenshot of https://example.com. The main session
should not have playwright tools in its tool list. Verify by running
/context in the main session after the subagent completes.
```

### Step 5 — Explore the Plugin Marketplace

```text
Install the claude-code-setup plugin via /plugin install claude-code-setup.
Run it and report: what it recommends for this repo, which recommendations
you would accept, and which you would skip with a one-line reason each.
```

---

## Expected Claude Code Behaviour

MCP servers register tools that appear in Claude's tool list and consume context tokens. Scoping playwright to a subagent keeps the main session leaner. The claude-code-setup plugin surfaces repo-specific configuration suggestions rather than generic advice.

> **Token Overhead:** Every MCP server adds tool-registration tokens to your context on every session start — even if you never use those tools. Scope rarely-used servers to subagents to keep the main context lean.

### Success Criteria

- All three MCP servers are added and their tool counts are reported.
- You have a before/after token delta showing MCP overhead cost.
- A real GitHub PR list is returned using MCP tools, not shell commands.
- Main session `/context` confirms playwright tools are absent after subagent scoping.
- claude-code-setup recommendations are reviewed with accept/skip decisions documented.

> **Reflection:** MCP servers provide tool access but also add token overhead on every session start. What is the right heuristic for deciding whether an integration belongs as an always-on MCP server versus a subagent-scoped tool versus a simple bash command in a hook?