# Runtime Environment — Digital Employee Sessions

This document describes how a Digital Employee (DE) session works and which tools are available inside an OpenClaw agent session.

---

## How a DE Session is Triggered

Every DE session is started by one of four trigger types:

| Trigger | Description | Example |
|---------|-------------|---------|
| **cron** | Scheduled execution via OpenClaw cron | Daily KPI measurement at 07:00 |
| **user** | Direct message from the operator | "What's the SoAV score today?" |
| **decision** | Escalation from another DE or scheduled task | CFO-DE asks GEO-DE: "approve budget?" |
| **colleague** | Multi-agent spawn from a Chief-DE | Chief delegates a sub-task to a specialist |

At session start, the DE reads its `job.md`, runs `pre_fetch.py` to measure KPIs, then decides what action to take based on goals and authority.

---

## Available Tools by Category

### Shell & Files — Core Execution Layer

These are the most important tools. Most DE logic runs through `exec`.

| Tool | Use | Notes |
|------|-----|-------|
| `exec` | Run shell commands, Python scripts, curl, systemctl | Primary workhorse. Use for all system-level actions. |
| `read` | Read a file from the workspace | Fast, no shell needed for simple reads |
| `write` | Write/overwrite a file | Creates parent dirs automatically |
| `edit` | Exact surgical edits to a file | Preferred over write for partial updates |
| `apply_patch` | Apply a unified diff patch | Good for multi-file changes |

**exec is the DE's Swiss Army knife.** Use it to:
- Run `pre_fetch.py` for KPI measurement
- Call external APIs via `curl`
- Run Python analysis scripts
- Manage processes via `systemctl`
- Read/write structured data (JSON, CSV)

```bash
# Example: measure and parse a KPI
exec: python3 /var/de-agents/<de-name>/framework/pre_fetch.py
exec: curl -s "https://api.example.com/metric" | python3 -c "import json,sys; print(json.load(sys.stdin)['value'])"
```

### Memory — Persistent State Between Sessions

DEs are stateless between sessions. Memory tools provide continuity.

| Tool | Use |
|------|-----|
| `memory_search` | Semantic search across MEMORY.md + memory/ files |
| `memory_get` | Read exact lines from a known memory file |

**Pattern:** Write decisions and KPI snapshots to `memory/YYYY-MM-DD.md` during execution. Use `memory_search` at session start to recall relevant prior context.

```bash
# Write to memory via exec
exec: echo "## KPI snapshot\n- SoAV: 42%\n- Leads: 5" >> memory/$(date +%Y-%m-%d).md
```

### Web Research

| Tool | Use |
|------|-----|
| `browser` | Full browser automation — Perplexity research, competitor analysis, web scraping |
| `pdf` | Parse and analyze PDF documents |
| `view_image` | Load an image into model context for visual analysis |

**Note:** DEs cannot directly access the internet via code. All HTTP requires either `exec` + `curl` (for APIs) or the `browser` tool (for web pages).

### Multi-Agent Coordination

| Tool | Use | When to use |
|------|-----|-------------|
| `sessions_spawn` | Launch a sub-agent session | Chief-DE delegating a specialist task |
| `sessions_send` | Send a message to a running session | Passing results between agents |

These tools are used by **Chief-DEs** that coordinate multiple specialist agents. See `docs/patterns/chief-agent.md`.

### Human Communication

| Tool | Use |
|------|-----|
| `message` | Send a message to the operator's messaging channel (Telegram, Discord, etc.) |
| `ask_user` | Ask the operator a structured question and wait for their answer |

**Critical rule:** Level-2 decisions (anything with external impact, budget, or irreversibility) MUST go through `message` or `ask_user`. The DE cannot self-approve these.

```python
# Level-2 decision pattern
message("🔔 Decision needed: I found 3 new leads. Should I send the outreach email? Reply YES/NO")
# Then wait — the response arrives as a new session trigger
```

### Workboard (Optional)

| Tool | Use |
|------|-----|
| `workboard_create` | Create a tracked task card |
| `workboard_claim` | Claim a card for execution |
| `workboard_complete` | Mark a card done with proof |

Use for multi-step experiments where you need audit trails and retry logic. Overkill for simple daily tasks.

### Media (Content DEs Only)

| Tool | Use |
|------|-----|
| `image_generate` | Generate images (product mockups, social graphics) |
| `tts` | Text-to-speech output |
| `video_generate` | Generate short video clips |

Only relevant for DEs whose mission involves content creation.

---

## What a DE Cannot Do

| Limitation | Reason |
|-----------|--------|
| No direct internet access in code | All HTTP must go through `exec`+`curl` or `browser` |
| Cannot self-approve Level-2 decisions | Human-in-the-loop required for irreversible actions |
| Cannot modify its own `job.md` without operator intent | job.md is the operator's contract with the DE |
| Cannot persist state in memory between `exec` calls | Use files or the workspace for intermediate state |
| Cannot spawn other DEs without Chief-DE pattern | Unauthorized spawning creates untracked agents |

---

## Execution Budget

Budget refers to the approximate number of reasoning/tool-call steps per session type.

| Session Type | Steps Budget | Description |
|-------------|-------------|-------------|
| cron (routine) | 5–10 | KPI measure → decision → message if needed |
| deep analysis | 10–20 | Research, competitor scan, full report |
| decision response | 3–8 | Human asked a question → answer it |
| experiment run | 15–30 | Multi-step test with verification |

Exceeding budget is a sign the DE's `job.md` scope is too broad. Split into specialist agents.

---

## OpenClaw Server Requirements

| Requirement | Detail |
|------------|--------|
| Python | 3.10+ |
| exec sandbox | **Not sandboxed** — runs as the OpenClaw process user |
| Messaging channel | Must be configured (Telegram, Discord, etc.) for `message` tool |
| Workspace | Must be writable; DE's workspace at `/var/de-agents/<de-name>/` |
| OpenClaw version | 0.9+ recommended for cron + messaging |

---

## Messaging Channel Setup

DEs send operator notifications via the configured messaging channel.

### Telegram Setup (Recommended)

Create `/etc/de-framework.env`:

```bash
# /etc/de-framework.env
TELEGRAM_BOT_TOKEN=bot<your-token>
TELEGRAM_CHAT_ID=<your-chat-id>
```

Source in cron or systemd service:

```bash
# In new-client-setup.sh or DE's cron entry
source /etc/de-framework.env
```

### Getting Your Chat ID

```bash
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getUpdates" \
  | python3 -c "import json,sys; updates=json.load(sys.stdin)['result']; print(updates[-1]['message']['chat']['id'])"
```

### Testing the Channel

```bash
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d chat_id="${TELEGRAM_CHAT_ID}" \
  -d text="✅ DE messaging channel active"
```

---

## DE Session Lifecycle

```
Session Start
    │
    ▼
1. Read job.md          ← mission, KPIs, authority levels
    │
    ▼
2. Run pre_fetch.py     ← measure current KPIs
    │
    ▼
3. Compare vs targets   ← on track? behind? blocked?
    │
    ├── On track → log + exit (HEARTBEAT_OK equivalent)
    │
    ├── Insight found → message operator with finding
    │
    ├── Level-1 action → execute autonomously + log
    │
    └── Level-2 decision → message operator, await response
```

A session that exits without messaging is a success if KPIs are on track. Silence = good. Noise = signal.

---

## DE Performance Benchmarks — Real-World Data

*From production runs, collected 2026-09-26.*

### Session Step Counts by DE Type

| DE Type | Examples | Typical Range | Healthy | Warning | Action |
|---|---|---|---|---|---|
| Monitoring | ops | 25-40 steps | ✅ complete | 40-55 steps | 55+ steps or max_iter |
| Security | shield | 30-55 steps | ✅ complete | 55-70 steps | 70+ steps or max_iter |
| Efficiency (admin) | flow, coach | 10-25 steps | ✅ complete | 25-35 steps | 35+ steps or max_iter |
| Research/GEO | grow_* | 10-30 steps | ✅ complete | 30-45 steps | 45+ steps or max_iter |
| Growth (experiments) | growth | 15-40 steps | ✅ complete | 40-55 steps | 55+ steps or max_iter |

### Warning: If cron session > 30 steps → STOP Block likely ineffective

When a scheduled session consistently hits 40+ steps, the STOP block is not working. Diagnose:

1. **Is `not_measured` in the briefing?** → `measure_soav.py` or measure command is reading from wrong path. Fix the path.
2. **Is the agent running benchmarks in-session?** → Move benchmark to `pre_fetch.py` via `run_benchmark_if_stale()`.
3. **Is the STOP block vague?** → Replace with specific file target + hard tool-call limit.
4. **Is there NO STOP block?** → Add one immediately. Every job.md must start with `## STOP`.

### Expected Completion Rate by Phase

| Phase | Target completion rate |
|---|---|
| Week 1-2 (bootstrap) | 40% — agents in benchmark mode, learning environment |
| Week 3-4 (early operation) | 60% — most KPIs measured, STOP blocks tuned |
| Month 2+ (production) | 80%+ — steady state |

Below 60% after month 2 = systemic problem (API keys, wrong paths, or vague STOP blocks).

