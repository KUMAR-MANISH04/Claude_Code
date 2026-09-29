# 29_E04_LLM-Powered Security

# E04 — Using LLMs to Secure Source Code

> A structured six-step methodology for deploying Claude Opus to find and fix vulnerabilities — threat modeling, sandboxed PoC verification, parallel discovery, adversarial triage, and patch validation. (Published May 27, 2026 — Eugene Yan & Henna Dattani)

---

## The Paradigm Shift: Discovery Is Solved, Verification Is the Bottleneck

The critical finding: **vulnerability discovery is now trivial to parallelize.** The bottleneck has shifted entirely to verification, triage, and patching.

**Field Evidence (May 2026):**

| Metric | Value |
| --- | --- |
| Vulnerabilities disclosed from open-source scanning | **1,596** |
| Patched as of May 22, 2026 | **97** |
| Exploitability rate when threat model is well-defined | **90%** |
| Reduction in false positives from adversarial verification | **50%** |

---

## The Six-Step Find-and-Fix Loop

### Step 1: Threat Model — Define the Attack Surface

The single biggest quality lever. Teams that skip it produce 40% false positives; teams with a well-defined threat model achieve 90% exploitability rates.

**Bootstrap from** existing architecture diagrams, git history, past CVEs, and Shostack's four questions:

- What are we building?
- What can go wrong?
- What are we doing about it?
- Did we do a good job?

**Deliverable:** `THREAT_MODEL.md` committed to the repo. Document trusted inputs, components, and dependency security policies explicitly.

### Step 2: Sandbox — Isolate and Enable PoC Verification

Two purposes: protect production from agent actions, and *prove* findings are actually exploitable. Unverified findings create triage debt.

```
Security requirements:
  - Container or microVM with strong isolation
  - Locked egress except to model API endpoint
  - No production credentials available to agents
  - Network access only during dependency setup phase

Environment consistency:
  - Pin all dependencies, commit SHAs, and image tags
  - Cache dependencies locally (no runtime downloads)
  - Mirror production configuration faithfully
  - Pin Docker dependencies to production-matching versions
```

> **Field Insight:** "The biggest efficacy lever has been giving the model test beds, live systems, and running the PoCs." PoC execution *is* the validation — without it, verification becomes the permanent bottleneck.

### Step 3: Discovery — Parallelize with Rich Context

Counterintuitive: **simpler prompts outperform detailed checklists.** Provide context and goals; let the frontier model determine its own methodology.

```
Input to each discovery agent:
  - THREAT_MODEL.md
  - Architecture documentation
  - Prior static analysis scan results
  - Target partition (by attack surface, endpoint, or component)

Output requested:
  - Structured vulnerability report (predefined fields)
  - Proof-of-concept code (if sandbox available)

Parallelization:
  - Spawn one agent per partition
  - Use disjoint partitions to avoid duplicate findings
  - More agents on the same target = diminishing returns + duplicates
```

### Step 4: Verification — Separate Discovery from Precision

This step alone roughly **halves false positive rates.** Key principle: the verifier must not know the discoverer's reasoning.

```
Verifier agent receives:
  - The finding only (no discovery agent context)
  - Access to the codebase

Verifier instructions:
  - "Try to DISPROVE this finding"
  - Search for compensating controls the finder missed
  - Attempt to reproduce the PoC in the sandbox

Multi-verification:
  - Deploy 3 independent verifiers
  - Majority vote (2/3) determines finding validity
  - Any verifier that reproduces PoC = finding is confirmed
```

### Step 5: Triage — Deduplicate and Prioritize

Run in two phases. Without the original threat model context, models overestimate severity — always give the triage agent the same threat model the discovery agent received.

**Phase A: Deduplication by Root Cause**

- Deterministic pass: same file + category + within 10 lines = duplicate.
- Qualitative assessment: does one patch disarm multiple PoCs?

**Phase B: Severity Rating**

| Dimension | Critical/High | Medium | Low |
| --- | --- | --- | --- |
| Preconditions | Zero | 1–2 | 3+ |
| Authentication | Unauthenticated remote | Authenticated | Local only / Admin |
| Impact Scope | Infrastructure / multi-tenant | Multi-user | Single user session |
| Attacker Control | Full control of vulnerable input | Partial control | Indirect influence |

### Step 6: Patching — Close the Loop

Agents tend to address symptoms rather than root causes, and generate overly restrictive patches. Enforce a strict quality ladder.

```
Gate 1: Build and compilation success
Gate 2: Original PoC no longer reproduces in sandbox
Gate 3: Existing test suite passes (no regressions)
Gate 4: Fresh discovery agent confirms fix is comprehensive

Patch agent instructions:
  - Write a failing test first
  - Fix the root cause, not the symptom
  - Search for variants at call-site and class level
  - Keep patch minimal — no refactoring, no formatting changes
  - Provide iterative feedback via PoC re-execution
```

> **Common Failure Mode:** Patches that "are as restrictive as possible… to the point they break legitimate connections." Always validate patch behavior against representative positive-case traffic, not just the exploit PoC.

---

## Getting Started: First Cycle

```bash
# 1. Clone the reference harness
git clone https://github.com/anthropics/defending-code-reference-harness

# 2. Run the interactive quickstart
claude /quickstart

# 3. Bootstrap threat model from existing docs
claude /threat-model --from-docs ./docs --from-history ./CHANGELOG.md

# 4. Build sandbox environment (match production)
docker build -f Dockerfile.sandbox -t sec-sandbox:prod .

# 5. Run first discovery-verify-triage-patch cycle
claude /vuln-scan --partition auth --sandbox sec-sandbox:prod
```

> **First cycle expectations:** You'll find more vulnerabilities than expected. Budget significant triage time before committing to patching. The initial scan is always the largest; subsequent periodic scans find residuals and regressions.

### Ongoing Operations Model

- **Scheduled Scans** — run discovery weekly or biweekly; connect to CI/CD to trigger on significant code changes.
- **Event-Triggered Scans** — trigger on new bug bounty reports, CVE publications in your dependency tree, static analysis alerts, or major dependency updates.
- **Historical Pattern Mining** — analyze past CVEs and security commits. "What have people exploited in this codebase before?" is faster than blind discovery.

> **SME Insight:** Models remain stochastic even with the same prompt. Long tails of vulnerabilities persist — run multiple scans with varied agent instances rather than relying on a single scan. Each iteration finds something new.

> Source: *Using LLMs to secure source code* — Anthropic Blog, May 27, 2026 — Eugene Yan & Henna Dattani.