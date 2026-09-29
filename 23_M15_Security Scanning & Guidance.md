# M15 — Security Scanning & Guidance

> Find and fix vulnerabilities in *your* code — the Claude Security plugin (scan → read → patch) and the always-on Security Guidance plugin.

---

## Why This Chapter Exists

Sandboxing (Module 12) stops Claude from *doing* dangerous things. This chapter is the opposite: finding the dangerous things already in **your** code. Two plugins cover this, at two different moments.

| Plugin | When it works | What you do |
| --- | --- | --- |
| Claude Security | On demand — you run a deep scan | Run `/claude-security`, read the report, apply patches |
| Security guidance | Automatically — as Claude writes code | Nothing. It reviews and fixes in the background |

> **Mental model:** the first is a security *audit* you request; the second is a security *reviewer* looking over Claude's shoulder as it types.

---

## Part 1 — The Claude Security Plugin

A team of Claude agents maps your architecture, builds a threat model, hunts for vulnerabilities, and **independently reviews every finding** before writing the report. You turn the findings you care about into patches that you review and apply yourself. Nothing is ever applied automatically.

> **Flow:** `/claude-security` → *Scan codebase* → verifier agents confirm findings → `CLAUDE-SECURITY-RESULTS.md` → *Suggest patches* → `F1.patch, F3.patch …` → you `git apply`, one PR each.

### Before You Start

| Requirement | Detail |
| --- | --- |
| Claude Code | v2.1.154 or later, on a paid plan |
| Dynamic workflows | The scan orchestrates agents with them. On Pro, enable in `/config` → *Dynamic workflows* |
| Python | 3.9.6+ on `PATH` as `python3`. Standard library only — nothing gets installed |
| Git | Needed for change scans and patches. A **full** scan works with or without version control |
| OS | Linux, macOS, or Windows |

### Install the Plugin

```
/plugin install claude-security@claude-plugins-official
```

If the marketplace isn't found, run `/plugin marketplace add anthropics/claude-plugins-official` first, then retry. Activate in the current session without restarting:

```
/reload-plugins
```

The plugin adds **one** command, `/claude-security`, which opens a menu of three jobs: scan the codebase, scan a set of changes, or suggest patches. To uninstall: remove it from `/plugin`, or run `claude plugin uninstall claude-security`.

> **Tip:** the plugin works best in **auto mode**, so the scan's agents don't stop for a prompt at every step.

### The Happy Path: Scan → Read → Fix

1. **Open the menu** — run `/claude-security`, pick *Scan codebase*.
2. **Choose what to scan** — whole repository or a focused area (each with file count and relative cost). Answer *"I don't know"* to let it pick a sensible default.
3. **Confirm the run** — scans take a while and use significant tokens. Nothing runs until you confirm, and Claude Code must stay open until it finishes.
4. **Watch progress** — each stage is reported; full detail lives under `/workflows`.
5. **Turn findings into patches** — run `/claude-security` again, pick *Suggest patches*.
6. **Apply what you accept** — `git apply` each patch, one per pull request.

> You can also ask directly — `/claude-security scan my branch` — or in plain language: *"scan commit abc1234"*, *"fix finding F3"*.

### Scanning Single vs Large Repositories

| Situation | What to do |
| --- | --- |
| Small / medium repo | Scan the whole repository in one pass |
| Large repo | Scan **one area at a time** — pick a focused scope (API layer, auth code) |
| Just my changes | When your branch is ahead of its base, the menu offers to scan only that diff |
| A single commit / PR | Ask: *"scan commit abc1234"*, or scan an open PR (needs `gh` signed in) |

> The report's **coverage section** always states what was and wasn't examined. Change scans need git and only read **committed** code — commit or stash first, or run a full scan (which reads the working tree).

### Reading the Results

Every scan writes a timestamped `CLAUDE-SECURITY-<timestamp>/` directory into your repo:

| File | What's in it |
| --- | --- |
| `CLAUDE-SECURITY-RESULTS.md` | The report — each finding's ID (e.g. `F1`), impact, exploit scenario, severity, confidence, recommendation |
| `CLAUDE-SECURITY-RESULTS.jsonl` | The same findings, one JSON object per line (machine-readable) |
| `CLAUDE-SECURITY-REVISION-<commit>.json` | The revision stamp — which commit was scanned, at what effort, whether uncommitted changes were included, how thoroughly verified |

> That directory is the **only** change a scan makes, and it ships with its own `.gitignore`. Findings only appear **after** independent verifier agents confirm them. Scans are **non-deterministic** — scan regularly and use revision stamps to tie each report to the exact code it covered.

### Fixing Findings with Patches

Start with *Suggest patches*, or say *"fix finding F3"*. Each patch is drafted in a **scratch copy** of your repo, so source files stay untouched. An agent *independent of the one that wrote it* reviews the change — running your tests when present. A patch is written only when that review can vouch that the change:

1. addresses the one finding,
2. introduces no new vulnerability, and
3. leaves behaviour otherwise unchanged.

If it can't vouch for all three, you get a short note explaining why instead of a patch.

> **Applying is always your call.** Patches land in `patches/`, one `F<n>.patch` per finding. Apply each in **its own pull request**.

```bash
git apply CLAUDE-SECURITY-<timestamp>/patches/F1.patch
```

Patches build against committed code — if the code changed since the scan, that finding is skipped with a note and you're offered a fresh scan.

> **Plugin vs managed product:** the plugin runs **locally in your session** and counts against your plan's usage limits. The managed **Claude Security** product (Enterprise) monitors connected repos. The plugin's advantage: it reaches code the managed product can't (GitLab/Bitbucket, or closed networks).

---
## Part 2 — The Security Guidance Plugin

Where Claude Security is an audit you *run*, security guidance is a reviewer that's *always on*. It makes Claude review its own changes for common vulnerabilities — injection, unsafe deserialisation, unsafe DOM APIs — and fix them **in the same session**, before the code reaches a PR. Once installed, it runs automatically. **There is nothing to invoke.**

```
/plugin install security-guidance@claude-plugins-official
/reload-plugins
```

Choose **user scope** when prompted so it loads in every new session. Needs Claude Code v2.1.144+ and Python 3.7+ (commit review wants 3.10+). On first run it creates a virtualenv under `~/.claude/security/`. To enable it for **everyone who clones a repo**, declare it in checked-in settings:

```json
{
  "enabledPlugins": {
    "security-guidance@claude-plugins-official": true
  }
}
```

### What It Checks — Three Layers

| Layer | Depth | Catches |
| --- | --- | --- |
| On each edit | Fast string match, no model call, no cost | `eval(`, `pickle`, `.innerHTML =`, `.github/workflows/` edits |
| End of turn | Background model review of the turn's diff | Authorisation bypass, IDOR, injection, SSRF, weak crypto |
| On commit / push | Agentic review reading callers & sanitisers | Real-vs-safe judgement with full surrounding context |

> Findings reach Claude as instructions; Claude fixes them and you see both the finding and the fix. **No layer blocks a write or commit** — treat it as one layer of defence, not a guarantee.

### Tailoring It to Your Repo

Add a threat model in `.claude/claude-security-guidance.md` (plain language, loaded as extra context):

```markdown
# Security guidance for this repo
- Do not log `customer_id` or `account_number` at INFO level or above.
- All routes under `/admin` must call `require_role("admin")` before any DB read.
- Use `crypto.timingSafeEqual` for token comparison instead of `===`.
```

Add deterministic string/regex rules in `.claude/security-patterns.yaml`:

```yaml
patterns:
  - rule_name: internal_api_key
    substrings: ["sk_live_", "AKIA"]
    reminder: "Hardcoded API key prefix. Load credentials from the secret manager."
```

> Both are **additive** — you can add checks but can't disable built-in ones from these files. To disable a whole layer, set an env var (`ENABLE_STOP_REVIEW=0`, `ENABLE_COMMIT_REVIEW=0`, or `SECURITY_GUIDANCE_DISABLE=1` for the lot).

### Using Both Together

- **Security guidance** shrinks the number of issues you ever write down, catching them as Claude codes.
- **Claude Security** is your periodic deep audit of everything already in the tree — including code written before the guidance plugin was installed.

> **Cadence:** run guidance always; run a Claude Security scan before releases, after big features, and on a regular schedule.

---

## How These Fit With Other Security Tools

Neither plugin replaces your existing scanners — they're layers in a defence-in-depth stack:

| Stage | Tool | What it covers |
| --- | --- | --- |
| In session | Security guidance plugin | Common vulnerabilities in code Claude writes, fixed the same session |
| On demand, single pass | `/security-review` | One-time security pass on the current branch |
| On demand, deep scan | Claude Security plugin | Multi-agent scan of a repo or diff, independently reviewed findings + patches |
| On pull request | Code Review (Team/Enterprise) | Multi-agent correctness + security review with full codebase context |
| Managed | Claude Security product (Enterprise) | Hosted scanning that monitors connected repositories |
| In CI | Your existing static analysis & dependency scanners | Language-specific rules, supply-chain checks, policy enforcement |

> The plugins reason about your code the way a human security researcher would — **complementing** the deterministic checks CI scanners provide.

---

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `/claude-security` menu opens with a Python warning | Needs `python3` 3.9.6+ on `PATH`. Install or reorder `PATH`, then start a new session |
| Security guidance reviews never appear | Not a git repo; or no Anthropic auth; or `security-patterns.yaml` exists but PyYAML isn't importable — use `.json`. Diagnostics: `~/.claude/security/log.txt` |
| Marketplace not found | `/plugin marketplace add anthropics/claude-plugins-official`, then retry |

---

## Check Your Understanding

> **Scenario:** You inherit a 200k-line service with no security history. You install the security guidance plugin, and it stays quiet for a week of feature work. A teammate asks why it hasn't flagged the SQL injection everyone suspects is in the legacy payment module. What do you tell them, and what do you run?

**Answer:** The guidance plugin only reviews code **Claude writes in-session** — it never scanned the legacy module because nobody edited it. Run a **Claude Security** deep scan instead: `/claude-security` → *Scan codebase*, scoped to the **payment module** rather than the whole tree. Read `CLAUDE-SECURITY-RESULTS.md`, then *Suggest patches* for any confirmed finding and `git apply` each in its own PR.