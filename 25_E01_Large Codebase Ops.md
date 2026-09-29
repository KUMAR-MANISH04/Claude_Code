# 26_E01_Large Codebase Ops

# E01 — Claude Code in Large Codebases

> Best practices for monorepos, legacy systems, and multi-service architectures — from Anthropic's official enterprise guidance (Published May 14, 2026).

---

## How Claude Code Navigates a Codebase

Claude Code navigates codebases the way an experienced engineer does on their first day: it traverses the file system, reads files, uses grep to find exactly what it needs, and follows references. This is fundamentally different from RAG-based systems that pre-embed everything.

| ✔ Claude Code (Local, On-Demand) | ✗ RAG-Based Systems (Centralized Index) |
| --- | --- |
| No pre-indexing required | Requires up-front embedding pipeline |
| Always reads current files | Embeddings go stale on active teams |
| Works with any repo structure | Misses recent commits and branches |
| No stale embedding problem | Centralized infrastructure cost |
| Runs on the developer's machine | Index refresh lag hurts reliability |

> **Key Principle:** The model's intelligence matters less than the infrastructure around it. The "harness" — CLAUDE.md files, hooks, skills, MCP servers, and how the team organizes them — determines Claude Code's effectiveness more than the model itself.

---

## The Seven-Component Harness

Anthropic identifies seven extension points that collectively form the "harness" — the primary leverage point for enterprise teams.

1. **CLAUDE.md Files** — Context loaded at session start. Root file provides overview; subdirectory files address local conventions.
2. **Hooks** — Scripts that trigger at key moments (pre-edit, post-turn, pre-commit) for automation, self-improvement, and policy enforcement.
3. **Skills** — On-demand expertise packages loaded only when invoked, keeping the base context lean.
4. **Plugins** — Bundled configurations distributed org-wide; one `/plugin install` gives everyone the same skills, hooks, MCP, and subagents.
5. **LSP (Language Server Protocol)** — Symbol-level navigation for typed languages: resolve types, jump to definitions, find references.
6. **MCP Servers** — Connections to internal tools and data sources (ticket systems, CI, internal docs, proprietary APIs).
7. **Subagents** — Isolated Claude instances for parallel exploration and editing without context contamination.

---

## Making Large Codebases Navigable

The most impactful decision is how you structure CLAUDE.md files. The recommended pattern is **lean and layered**:

```
repo-root/
├── CLAUDE.md              # ← High-level overview, global conventions, owners
├── services/
│   ├── auth/
│   │   └── CLAUDE.md      # ← Auth-service specific: JWT strategy, test commands
│   ├── payments/
│   │   └── CLAUDE.md      # ← Payment-service: PCI boundaries, test isolation
│   └── notifications/
│       └── CLAUDE.md      # ← Notification service conventions
├── packages/
│   ├── shared-ui/
│   │   └── CLAUDE.md      # ← Component standards, design system imports
│   └── utils/
│       └── CLAUDE.md      # ← Utility library conventions
└── .claudeignore          # ← Exclude generated code, vendor, dist/
```

### What Belongs in Each Layer

| File | Contents | Avoid |
| --- | --- | --- |
| `root/CLAUDE.md` | Tech stack, global lint/test commands, team contacts, architecture overview, security policies | Service-specific details that will age quickly |
| `service/CLAUDE.md` | Service purpose, local test commands, key patterns, known gotchas, data model overview | Repeating root-level info already loaded |
| `package/CLAUDE.md` | Public API contract, contribution rules, versioning policy | Implementation details derivable from the code |

> **Don't initialize at repo root for monorepos.** Starting inside the specific service directory loads only relevant CLAUDE.md files and scopes tool access correctly. Root-level init on a 500k-line monorepo overwhelms the context.

---

## The .claudeignore Pattern

Like `.gitignore`, a `.claudeignore` file prevents Claude from reading irrelevant content, preserving context for code that matters.

```
# Generated code — never edit manually
dist/
build/
*.generated.ts
**/generated/
*_pb.ts        # protobuf-generated files

# Third-party vendored code
vendor/
node_modules/
.pnpm/

# Large binary and data assets
*.png
*.jpg
*.pdf
data/
fixtures/*.json   # large test fixtures

# Infrastructure definitions rarely needing code context
terraform/
k8s/
helm/
```

> **SME Tip:** Run `claude /count-tokens` before and after adding `.claudeignore` entries. A well-tuned `.claudeignore` on a mature monorepo typically cuts context by 40–70%.

---

## Configuration Maintenance Lifecycle

CLAUDE.md entries that once helped can become unnecessary or *actively constraining* when a new model ships. Enterprise teams need a schedule:

- **After a major model release** — audit every entry added as a workaround for model behavior; test whether the new model already handles it; remove what's no longer needed.
- **Every 3–6 months (routine)** — review CLAUDE.md against recent PRs; remove entries for changed patterns; add entries for new patterns needing correction; keep files under 500 words each.
- **On every sprint (hooks approach)** — a hook mines recent sessions for repeated corrections and proposes CLAUDE.md additions automatically, creating a self-improving harness.

---

## Organizational Structure for Rollout

The most common failure mode is **enthusiastic bottoms-up adoption that fragments without central coordination**. Two patterns that work:

- **Pattern A: Dedicated Infrastructure Team** — a "developer experience"/"productivity engineering"/"agent management" team owns the shared harness config, plugin distribution, and MCP catalog. *Best for: organizations >200 engineers.*
- **Pattern B: Single DRI (Directly Responsible Individual)** — one senior engineer owns configuration authority across teams, with write access to the shared plugin, CLAUDE.md templates, and hook catalog; teams propose via PR. *Best for: fast-moving startups and mid-size teams (<200 engineers).*

> **What the DRI/Team does weekly:** review sessions where Claude repeatedly corrected the same way; consolidate effective CLAUDE.md snippets into the shared plugin; curate the MCP catalog; monitor token costs and adjust `.claudeignore`.

---

## SME Assessment: Are You Ready?

- ☐ Root CLAUDE.md exists and is under 500 words
- ☐ Service/package CLAUDE.md files are layered
- ☐ `.claudeignore` excludes generated and vendor code
- ☐ LSP server configured for primary languages
- ☐ A DRI or infra team owns the shared config
- ☐ CLAUDE.md review cadence is scheduled
- ☐ Teams initialize in service subdirs, not repo root
- ☐ MCP servers connect to internal tools/docs

> Source: *How Claude Code works in large codebases: Best practices and where to start* — Anthropic Blog, May 14, 2026.