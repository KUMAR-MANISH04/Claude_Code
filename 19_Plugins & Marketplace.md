# 19_Plugins & Marketplace

# M14 — Plugins & Marketplace

> `plugin.yaml` schema, 170+ plugins, Security Guidance Plugin deep-dive, building your own, and distribution.

---

## Why Plugins Exist

Skills, hooks, MCP servers, and subagents are powerful — but distributing them across a team is friction-heavy. Plugins package all four primitives into a single installable unit:

```
/plugin install security-guidance@claude-plugins-official
```

Team members who open any project automatically inherit the installed plugin's skills, hooks, MCP connections, and subagents.

---

## plugin.yaml Schema

```yaml
name: security-guidance
version: 1.2.0
description: Three-layer security review on every code change
author: Anthropic
scope: project          # "user" or "project"
requires:
  claude-code: ">=2.1.144"
  python: ">=3.8"

skills:
  - skills/security-review.md
  - skills/vulnerability-scan.md

hooks:
  - hooks/pre-edit-pattern-check.sh
  - hooks/post-turn-diff-review.sh
  - hooks/pre-commit-deep-review.sh

subagents:
  - subagents/security-reviewer.md

mcp:
  - mcp/semgrep-server.json
```

### Directory Structure

```
my-plugin/
├── plugin.yaml               # Manifest (required)
├── skills/                   # Skill markdown files
│   └── security-review.md
├── hooks/                    # Hook scripts (Bash, Python)
│   ├── pre-edit-pattern-check.sh
│   └── post-turn-diff-review.sh
├── subagents/                # Subagent definitions
│   └── security-reviewer.md
└── mcp/                      # MCP server configs
    └── semgrep-server.json
```

### Scope: User vs Project

| Scope | Where Installed | Who Gets It |
| --- | --- | --- |
| `user` | `~/.claude/plugins/` | That developer, in every project |
| `project` | `.claude/plugins/` | Everyone who opens that project |

> Most team-facing plugins use `project` scope and are committed to the repository.

---

## Plugin Marketplace (170+ Plugins)

```bash
# Install a specific plugin
/plugin install {name}@claude-plugins-official

# Browse interactively
/plugin > Discover
```

### Categories

| Category | Count | Examples |
| --- | --- | --- |
| Anthropic-built | 33 | security-guidance, claude-code-setup, code-review, test-runner |
| Partner | ~100 | github, playwright, supabase, figma, vercel, linear, sentry, stripe |
| Community | 40+ | Various — review hook scripts before installing |

### Key Partner Plugins

| Plugin | Adds |
| --- | --- |
| `github` | PR creation, issue triage, branch management via MCP |
| `playwright` | Browser automation and visual testing |
| `supabase` | Database schema review, migration safety checks |
| `figma` | Design token extraction, component comparison |
| `linear` | Issue tracking, sprint board updates |
| `sentry` | Error trace analysis, alerting integration |

---

## Security Guidance Plugin — Deep Dive

Released May 2026. Adds a three-layer security review to every session without requiring a separate CI step or model call on every edit.

> **Outcome:** 30–40% decrease in security-related PR comments in teams that have deployed it.

```bash
/plugin install security-guidance@claude-plugins-official
```

**Requirements:** Claude Code ≥ 2.1.144, Python ≥ 3.8.

### Three-Layer Architecture

**Layer 1: Fast Deterministic Pattern Match** — runs on every file edit. No model call — pure regex/AST matching. Catches ~25 dangerous patterns:

| Category | Patterns Caught |
| --- | --- |
| Code execution | `eval()`, `new Function()`, `exec()`, `compile()` |
| OS access | `os.system()`, `os.popen()`, `subprocess.call(shell=True)` |
| Child process | `child_process.exec()`, `child_process.spawn()` with shell |
| Deserialisation | `pickle.loads()`, `yaml.load()` without `Loader=`, `marshal.loads()` |
| DOM injection | `innerHTML =`, `outerHTML =`, `document.write()` |
| React unsafe | `dangerouslySetInnerHTML`, `__html` prop |
| SQL | String-concatenated queries (basic pattern) |
| Hardcoded secrets | Entropy-based token detection |

**Layer 2: Comprehensive Diff Review** — runs after each model turn. Reviews the full diff's semantic meaning. Flags privilege-escalation logic errors, missing input validation on new parameters, insecure defaults in config changes, and race conditions.

**Layer 3: Deep Review on Commit / Push** — triggered on `git commit` or `git push`. Loads surrounding files, checks sanitiser coverage, traces related code paths. Most expensive layer, runs infrequently.

### Configuration

```json
{
  "plugins": {
    "security-guidance": {
      "layer1": { "enabled": true, "customPatterns": ["my_unsafe_fn"] },
      "layer2": { "enabled": true },
      "layer3": { "enabled": true, "blockOnCritical": true }
    }
  }
}
```

> `blockOnCritical: true` prevents commits with critical findings from proceeding until the engineer explicitly overrides.

---

## Claude Code Setup Plugin

Analyses your codebase and recommends a tailored extension stack.

```bash
/plugin install claude-code-setup@claude-plugins-official
```

| Signal | Recommendation |
| --- | --- |
| `package.json` with React/Next.js | Playwright MCP for browser testing |
| Auth-related files | security-reviewer subagent |
| Database schema files | supabase or postgres MCP |
| `.github/workflows/` present | github plugin + CI skill |
| Large codebase (>50k lines) | smart-explore subagent |
| Python project | Python LSP plugin |

---

## Building Your Own Plugin

```yaml
name: my-company-standards
version: 0.3.1
description: Enforces engineering standards across all projects
author: platform-team@mycompany.com
scope: project

requires:
  claude-code: ">=2.1.0"

skills:
  - skills/pr-checklist.md
  - skills/migration-review.md
  - skills/incident-response.md

hooks:
  - hooks/enforce-conventional-commits.sh
  - hooks/block-prod-deploy-on-feature.sh

subagents:
  - subagents/platform-oncall.md

mcp:
  - mcp/internal-github.json
  - mcp/datadog.json

readme: README.md
```

### Team Distribution

| Method | When to Use |
| --- | --- |
| Private marketplace | Enterprise — managed via Claude Console |
| Direct git URL | Small teams: `/plugin install git+ssh://github.com/myorg/plugin.git` |
| Local path | Development: `/plugin install ./plugins/my-plugin` |
| Committed to repo | Check into `.claude/plugins/my-plugin/` — no install step needed |

> **Simplest distribution:** Commit the plugin directory to `.claude/plugins/`. Every developer who clones the repo gets it automatically.

---

## When to Use Plugins vs Skills vs Hooks

| Situation | Recommended Approach |
| --- | --- |
| Extending one project, one person | Skill or hook file in `.claude/` |
| Sharing a workflow with the team | Skill committed to repo |
| Adding MCP server + related skills together | Plugin |
| Enforcing security rules organisation-wide | Plugin via managed settings |
| Temporary task-specific automation | Hook file — no install overhead |
| Partner service integration | Official partner plugin |
| Replacing multiple fragmented configs | Plugin to unify them |

> **Decision heuristic:** If the feature involves more than one primitive (hook + skill + MCP), package it as a plugin. If it's a single markdown skill, keep it as a skill file.

---

## Common Mistakes

- **Installing too many plugins** — each plugin's MCP servers add tool definitions to every API request. Five MCP-heavy plugins can add 8–12k tokens of overhead per turn. Audit with `/mcp`.
- **Not reviewing plugin permissions** — hook scripts run with your credentials and full access to your working directory. Verify with `/plugin info <name> --show-hooks`.
- **Using community plugins without reviewing hook scripts** — community plugins are not vetted by Anthropic. A malicious `PostToolUse` hook on `Write` could exfiltrate file contents. Always read hook scripts first.
- **Assuming scope inheritance** — a `user`-scope plugin installed by one developer is invisible to others. Team-wide standards must be `project`-scope or distributed via managed settings.