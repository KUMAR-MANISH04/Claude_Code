# M16 — Maximising the Value of CC Sessions

> You pay for tokens. A small number of habits control most of that cost. The one mechanism that explains them all is the **prompt cache**.

---

## Why This Chapter Exists

Two sessions can do the same work and cost very different amounts. The difference is rarely the model — it's how you shape the session: what you load, how long you keep it, and what you change mid-session.

Every rule comes from one fact: **Claude Code sends your whole conversation on every turn.** Anything sitting in the conversation is paid for again, every turn, until you remove it.

> The prompt cache makes that repeat cost small — but only while the cache holds. Most waste comes from breaking the cache without knowing it.

---

## The Six Rules

| Rule | Why it works |
| --- | --- |
| Run `/clear` between tasks | The old task isn't needed. Without `/clear`, you pay for it on every later turn. |
| Set your model and effort level **before** you start | A mid-session change breaks the cache. The whole conversation is charged again. |
| @-mention a file instead of naming it | Claude receives the file directly — no search-then-read. |
| Make commands quiet, or run them in a subagent | Command output stays in the conversation for the rest of the session. |
| Run `/context` once in a fresh session | It shows what's loaded before you type. Remove what you don't use. |
| Run `/compact` before a long break | The cache expires after one hour. Compact while still cached and cheap. |

---

## What Makes a Token Expensive

| Factor | Effect | What you control |
| --- | --- | --- |
| The model | A larger model costs more per token | Choose the smallest model that does the job |
| The token type | Output tokens cost ~**5×** input tokens (generated one at a time) | Ask only for the work you need. Long output is expensive |
| The cache state | A cached token costs **0.1×** input price; a cache write costs up to **2×** | Keep the cache. The largest lever you have |

---

## How the Prompt Cache Works

The cache is automatic — you can't switch it on, but you can break it. Claude Code always builds the request in the **same order**: tool definitions → system prompt → conversation (with `CLAUDE.md` at the front). That fixed order makes caching possible.

The rule: *if a request starts with exactly the same tokens as one the server just saw, the result for that shared beginning is the same.* The server keeps that state and computes only what comes after. So the server reads down from the top, compares line by line, reuses everything up to the first difference (0.1×), then recomputes from there to the bottom at full price (and rewrites the cache at up to 2×).

### What Breaks the Cache

| Action | Effect on the cache | Do this instead |
| --- | --- | --- |
| `/model` switch | Every model has its own cache. Whole conversation refilled at full price | Choose the model before starting |
| `/effort` change | Effort is part of the cache key. Same full refill | Choose effort before starting |
| Fast mode on/off | Also part of the cache key. Same full refill | Decide once, at the start |
| `/compact` | Rewrites the conversation, so the cached version no longer matches | Still use it — but while the old conversation is *still cached*, so the summary is cheap |
| Time | Cache expires after **1 hour** (subscription) or **5 minutes** (API key). Set `ENABLE_PROMPT_CACHING_1H=1` to make an API key one hour | Finish in one sitting, or `/compact` before the break |
| Resuming an old session | Cache has almost always expired; first turn pays for the whole history | Resume only when you need the history |

### What Keeps the Cache

- **Appending to the end** — a tool result added to the end is ideal; nothing sits behind it.
- **Using `/rewind` instead of `/compact`** — rewind removes recent turns and leaves the earlier prefix untouched, so it costs nothing.
- **Not changing settings mid-task** — model, effort, and fast mode are all part of the cache key.

> **Rule of thumb:** add to the end, never edit the beginning.

---

## What Fills Your Context Window

The context window fills in three stages:

1. **Before you type** — tool definitions, system prompt, `CLAUDE.md`, and every connected MCP server.
2. **While Claude works** — every file it reads.
3. **After each command** — command output stays for the rest of the session, unless larger than 30,000 characters.

| Habit | How to do it |
| --- | --- |
| @-mention files | Type `@src/auth/login.ts` instead of describing it — skips the search and the Read call |
| Make commands quiet | Add the quiet flag, e.g. `npm test --silent` or `pytest -q` |
| Write the flags into `CLAUDE.md` | Record each routine command *with* its quiet flags so Claude always uses the quiet form |
| Run noisy jobs in a subagent | Log processing, large test suites, repeated high-output tasks — a subagent returns only a summary |
| Audit the start-up load | Run `/context` in a fresh session; it lists what's loaded before you type |
| Disconnect unused MCP servers | Run `/mcp`; every connected server adds tool definitions to every turn |

---
## Session Shape: One Long Session Costs More

One long session costs more than the same work spread over a few short ones — by more than most people expect. Turn 30 carries turns 1–29 with it; turn 1 of a fresh session carries nothing. (Worked example: over 24 turns, one long session costs ~3.6× the same work split with `/clear` every 6 turns.)

### Choose the Right Command

| Command | Use it when | Cost |
| --- | --- | --- |
| `/clear` | Starting a **different** task; the old context has no value | Free. Next turn starts a new, small cache |
| `/compact` | You need the history but the session is long, or you're about to break for >1 hour | Reads and summarises, then the cache refills. Do it while still cached |
| `/rewind` | The last few turns went wrong and you want to cut them | Nothing. The earlier prefix is unchanged, so it stays cached |
| Subagent | One job produces a lot of output the main session doesn't need | You pay for the subagent's turns, but its output never enters your context |

> **Before a break:** run `/compact` *before* you walk away, not after. Before the break the old conversation is still cached, so the summary is cheap. After the hour, it isn't.

---

## Model and Effort Level

Choose both **before** the task, because changing either mid-session breaks the cache.

- **The model** sets how much Claude *knows* — how deep its pattern recognition goes.
- **The effort level** sets how much Claude *does* — files read, verification, how far it pushes multi-step work before checking in.

**When Claude got it wrong:** first check the context (was the prompt clear? were the right files/tools available?). Then ask one question — did it not *know* enough, or not *try* enough?

- **Didn't know enough** (had every file, clearly tried, still wrong) → **move up a model**.
- **Didn't try enough** (skipped a file, didn't run tests, stopped halfway) → **raise the effort level**, same model.
- **Don't change both at once** — you won't learn which was the problem, and you pay two full cache refills.

### Which Model

| Model | Think of it as | Use it for |
| --- | --- | --- |
| Fable | The specialist for the problem everyone's stuck on | Genuinely hard problems, subtle bugs, unfamiliar domains, architecture, long multi-step work |
| Opus | The expert with broad experience | Complex work needing deep pattern recognition, but not open-ended research |
| Sonnet | A very good generalist | Most work. Well-defined tasks, routine edits, mechanical changes |
| Haiku | Fast and precise, with clear instructions | Simple, repetitive tasks where you can state exactly what to do |

### Which Effort Level

| Level | Behaviour | Choose it when |
| --- | --- | --- |
| Low | Prefers to ask you rather than spend tokens working it out | You know the answer and want fast, cheap execution |
| Default | Scales token use to what most people would want to spend | Almost always. Start here |
| High | Reads more files, verifies more, pushes through multi-step work (can generate ~**7×** more tokens for confidence) | Correctness matters more than cost, or Claude keeps stopping short |

> **Important:** effort shapes token use — it doesn't cap it. It's guidance the model learned to follow, not a hard limit.

> **Cost note:** a larger model isn't always more expensive overall. On hard work, a small model can grind through many failed iterations; a larger model that finishes in one pass can cost less in total.

---

## See It Live: The Status Line

The status line is a small script Claude Code runs after each turn and prints above your prompt — the instrument panel for the six rules.

**The two numbers that matter most:**

| Field | What it tells you | Act when |
| --- | --- | --- |
| `ctx 48250/200k 24%` | How full the context window is (green <50%, amber <80%, red ≥80%) | It turns amber — finish the task, then `/clear` |
| `(in … cache … out …)` | The split for the **last turn only**. `cache` = read/written to cache; `in` = full price | `cache` drops to near zero on a turn where you changed nothing → you broke the cache |

> **Watch during training:** note the `cache` number, run `/model`, send any message — `cache` stays large but is now a cache *write* and cost jumps. Run `/clear` and resend: everything is small again.

**Install it** (needs `jq`: `brew install jq` / `apt install jq`):

1. Save the script as `~/.claude/statusline.sh`.
2. `chmod +x ~/.claude/statusline.sh`.
3. Add to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh"
  }
}
```

Claude Code sends the script one JSON object on stdin after each turn; the script prints one or more lines. It's not a hook — it only reads and prints. Use `~/.claude/settings.json` for every project, or `.claude/settings.json` for one project.

---

## Your First Five Minutes in a Session

1. Run `/context`. Look at what's loaded before you type.
2. Run `/mcp` and disconnect servers this task doesn't need.
3. Set your model and effort level **now**. Don't change them later in this task.
4. @-mention the files you already know are involved.
5. Do one task. Then run `/clear` before the next.
6. Keep the status line in view. Watch `ctx` and the `cache` number after every turn.
7. Going for lunch? Run `/compact` before you leave.

---

## Check Your Understanding

> **Scenario:** An engineer keeps one session open all day. At ~turn 40 the answers feel weak, so they switch to a larger model. Still weak, so they raise the effort level too. They run `/compact` and take a one-hour lunch. Their spend is 4× a colleague's for similar work. Name four things they did wrong.

**Answer:**

1. **One session all day** — every turn carried the whole day's history. Should have run `/clear` at each new task.
2. **Switched model mid-session** — refilled the entire conversation at full price, at its largest point.
3. **Changed effort immediately after** — a second full refill, and now they can't tell which change helped.
4. **`/compact` before an hour away** — correctly timed, but the idle hour expired the new cache anyway. The real error was not clearing much earlier, so the conversation was huge by then.

**Better order:** check the context and prompt first. Then ask whether Claude didn't *know* enough or didn't *try* enough, and change **one** setting — in a fresh session, before the work starts.