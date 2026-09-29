# 03_Memory & CLAUDE

# Memory & CLAUDE.md

> The 4-layer memory system, CLAUDE.md at 3 scopes, auto-memory triggers and storage, memory types with YAML frontmatter, and the MEMORY.md index.

---

## Memory System Overview

Claude Code maintains four distinct layers of memory, each with different durability.

| Layer | Who Writes | Durability | Loaded |
| --- | --- | --- | --- |
| CLAUDE.md | You | Permanent | Every session |
| Auto-memory files | Claude | Permanent | Every session via index |
| Memory files | You (explicit save) | Permanent | Every session |
| Project memory | Claude | Per-session | Current session only |

---

## CLAUDE.md Files

CLAUDE.md is the foundation — you write it, Claude reads it at the start of every session.

### Three Scopes

| File | Scope | Loaded When |
| --- | --- | --- |
| `~/.claude/CLAUDE.md` | User-global | Every session, any project |
| `./CLAUDE.md` | Project | Any session in that project root |
| `./src/CLAUDE.md` | Directory | When Claude works inside `./src/` |

> Directory-scoped files load dynamically as Claude navigates. Place `./api/CLAUDE.md` with API conventions and it loads only when Claude touches the `api/` directory.

### 500-Line Discipline

> CLAUDE.md is loaded in full on every session. After ~500 lines, you are paying for context Claude may not need this session.

**Include**

- Build and test commands
- Linting and formatting rules
- Architecture decisions that affect every file (monorepo structure, module boundaries)
- Conventions Claude gets wrong without guidance (naming, file organisation)
- Links to important design docs

**Exclude**

- Workflow guides (move to skills — loaded on demand)
- Historical context about past decisions (move to auto-memory)
- Task-specific instructions (put them in the prompt)
- Documentation Claude can find by reading the code

### Example Project CLAUDE.md

```markdown
# Project: Payments Service

## Commands
- Build: `pnpm build`
- Test: `pnpm test -- --coverage`
- Lint: `pnpm lint --fix`

## Architecture
- Monorepo. Services in `packages/`. Shared types in `packages/shared/`.
- All DB access goes through `packages/db/src/client.ts` — never import Prisma directly.
- Event publishing: use `packages/events/src/publisher.ts`.

## Conventions
- Files: kebab-case. Types/interfaces: PascalCase. Functions: camelCase.
- No `any` in TypeScript. Prefer `unknown` + type guard.
- All new endpoints require a matching integration test in `tests/integration/`.

## Do Not Touch
- `packages/legacy-billing/` — frozen, owned by finance team.
```

---

## Auto-Memory

Claude's mechanism for persisting knowledge it discovers during a session — without you having to explicitly ask it to remember.

### Storage Location

```
~/.claude/projects/{project-hash}/memory/
├── MEMORY.md              # Index file (read at session start)
├── build-commands.md
├── debugging-session-2026-05-14.md
├── architecture-discovery.md
└── style-preferences.md
```

The `{project-hash}` is derived from the project's absolute path. Each project has its own memory directory.

### What Triggers Auto-Memory

| Trigger | Example |
| --- | --- |
| You correct Claude | "No, we use pnpm, not npm" |
| You confirm an approach works | "That fixed it — good approach" |
| Claude discovers something non-obvious | Finding an undocumented module boundary |
| Claude makes an architecture decision | Choosing between two valid patterns |
| A debugging session reveals root cause | Identifying a race condition |

> Claude does not save everything. It filters for durable facts that would genuinely help a future session — not ephemeral task state.

### What Gets Saved

**Good candidates:**

- Build quirks not documented anywhere (`GONOSUMCHECK=*` required for internal modules)
- Debugging insights ("The flaky test is caused by a shared timer in test setup, not the component")
- Architecture discoveries ("The canonical import is `auth/index.ts`")
- Style preferences you corrected Claude on ("Use `const` with type annotation, not `let` without")

**Not saved:** "Fix the login button" (task state), "The PR is ready" (ephemeral), standard patterns already documented in the codebase.

---

## Memory Types

Individual memory files use YAML frontmatter to declare their type and purpose.

```markdown
---
name: Build command discovery
description: Non-obvious build requirements for the payments service
type: project-context
created: 2026-04-01
---

`pnpm build` fails silently if `STRIPE_KEY` is not set, even in test mode.
Set it to `sk_test_placeholder` for local builds.

The `packages/legacy-billing/` build step is skipped automatically when
`LEGACY_BILLING=0` is set — required for engineers who don't have DB access.
```

### Type Reference

| Type | Purpose |
| --- | --- |
| `user-profile` | Preferences that apply across all projects |
| `feedback` | Corrections and style guidance |
| `project-context` | Project-specific architecture and conventions |
| `reference-pointer` | Links to key files or external docs |
| `debugging-insight` | Root causes discovered during debugging sessions |

---

## MEMORY.md Index

The index Claude reads at session start to decide which memory files are relevant. It is **not** loaded in full — Claude reads the index and loads individual files as needed.

> **Hard limit: 200 lines.** Content beyond 200 lines is truncated at session load. Keep the index lean.

```markdown
# Project Memory Index

## Session: 2026-05-14

- [Build command discovery](build-commands.md) — Non-obvious STRIPE_KEY requirement, legacy billing skip
- [Architecture discovery](architecture-discovery.md) — Auth module canonical imports
- [Style preferences](style-preferences.md) — const with type annotation, kebab-case file names

## Session: 2026-04-28

- [Debugging: flaky tests](debugging-2026-04-28.md) — Shared timer in test setup, fix applied
```

**Keeping the index healthy:**

- Remove entries for resolved issues (flaky test fixed → remove that entry).
- Consolidate duplicate entries after major refactors.
- If MEMORY.md approaches 200 lines, archive old entries to a separate file.

---

## Dreaming (Research Preview)

> Experimental as of May 2026.

When Claude Code is idle, agents can automatically review past session memories and refine them — merging duplicates, promoting important discoveries, deprecating stale entries. This is called **dreaming**.

```json
{
  "memory": {
    "dreaming": {
      "enabled": true,
      "schedule": "daily"
    }
  }
}
```

Expected behaviour: the MEMORY.md index stays coherent over time without manual curation. Contradictory memories are flagged for your review rather than silently resolved.

---

## Memory vs CLAUDE.md Decision

| What You Want to Store | Where |
| --- | --- |
| Build and test commands | Project CLAUDE.md |
| Linting rules and conventions | Project CLAUDE.md |
| Architecture decisions affecting all files | Project CLAUDE.md |
| Personal style preferences | User CLAUDE.md |
| Debugging insights from a specific session | Auto-memory |
| Non-obvious build quirks discovered at runtime | Auto-memory |
| Workflow guides (PR review process) | Skill file |
| Historical context about why a decision was made | Auto-memory |
| Current task state | Prompt (not memory) |
| Ephemeral notes | Prompt (not memory) |

> **Heuristic:** CLAUDE.md is for stable, always-relevant facts. Auto-memory is for things Claude learns through experience. Skills are for workflows. Prompts are for current tasks.

---

## Common Mistakes

- **Saving ephemeral task details** — Auto-memory is permanent; "we are currently refactoring the payment flow" is stale within a day. Apply the same filter when saving memory files explicitly.
- **Letting MEMORY.md grow past 200 lines** — Entries beyond the limit are invisible at session start. With more than ~80 entries, consolidate or archive.
- **Not capturing feedback after corrections** — The highest-value auto-memory comes from corrections. Confirm them explicitly so they're saved:

  ```
  That's right — always use const or let. Remember that for this project.
  ```

  Without confirmation, Claude may not save the correction.
- **Duplicating CLAUDE.md content in memory** — You pay for the context twice and risk the copies drifting out of sync.
- **Using a single global CLAUDE.md for all projects** — User-scoped `~/.claude/CLAUDE.md` loads for every project, so project-specific instructions there become confusing and potentially misleading.