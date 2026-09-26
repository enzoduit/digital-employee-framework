# DE Framework — Requirements & Quality Baseline

*Living document — update after every major iteration or health check.*

---

## Framework Requirements

### R1 — Session Efficiency by DE Type

| DE Type | Max Steps | Notes |
|---|---|---|
| Monitoring | ≤ 40 | Checking N services; completions are OK |
| Security/Audit | ≤ 55 | Deep scan requires more steps |
| Admin/Coordination | ≤ 25 | Briefing-driven; no exploration |
| Research/GEO | ≤ 30 | **Benchmark runs in pre_fetch, NOT in session** |
| Content | ≤ 40 | One focused page action per session |
| Analysis | ≤ 35 | Read briefing → compute → log |

`max_iterations_reached` = FAILURE state. Never acceptable as normal operation.

---

### R2 — Session Completion Rate

- **Target:** ≥ 80% sessions with status `complete`
- **Bootstrap (weeks 1-2):** 40% accepted
- **Early operation (weeks 3-4):** 60% target
- **Production (month 2+):** 80%+

Below 60% after month 2 = systemic problem. Investigate STOP blocks, API keys, and data paths.

---

### R3 — KPI Measurement

- Every DE must have ≥ 1 measurable KPI in `workspace/kpis.yaml`
- `measure` command must print one number or `not_measured` to stdout
- `not_measured` after 2nd session = failure signal → investigate measure command path
- **Common bug:** `measure_soav.py` reads from wrong directory → always returns `not_measured` → agent runs full benchmark in-session → 43-52 steps

---

### R4 — pre_fetch Pipeline

- `pre_fetch.py` MUST run in < 30 seconds (session_runner.py default timeout)
- If pre_fetch fails → session should abort cleanly (not run without briefing)
- **Research DEs:** `pre_fetch.py` must cache external query results (max 6h old) via `run_benchmark_if_stale()`
- Benchmark timeout: 300s (5min max); failure is non-fatal, session continues with cached data

---

### R5 — STOP Block Specificity (Critical)

Every DE `job.md` MUST have a `## STOP` section as the FIRST content block.

**Required elements in every cron STOP block:**
1. **Hard tool call limit:** "HARD LIMIT: max N tool calls total"
2. **Concrete DONE definition:** "DONE = [specific file] updated"
3. **Explicit DO NOT list:** the most common rabbit holes for this DE type

**Anti-patterns (do not use):**
- ❌ "max 5 iterations" — agent decides cron work requires more
- ❌ "take ONE focused action" — agent spends 20 steps deciding what that action is
- ❌ Missing STOP block entirely — guaranteed max_iterations_reached

**Working patterns:**
- ✅ "HARD LIMIT: max 4 tool calls. DONE = workspace/log.md updated. Stop after that write."
- ✅ "DO NOT run Perplexity queries in-session. pre_fetch.py already did it."
- ✅ "After N tool calls: write whatever you have to log.md and STOP."

---

### R6 — Service Monitoring DEs

- Must run on a fixed schedule (e.g., every 2h)
- All critical services must be in kpis.yaml with health check commands
- Internal OpenClaw services (canvas-server etc.) = explicitly excluded
- Sessions should complete, not max_iterate

---

### R7 — Human Decision Channel

- Level-2 decisions must reach a portal inbox or equivalent
- Delivery: Portal-first (Decisions API)
- Telegram/chat: secondary, urgent/real-time only

---

## Health Check Commands

```bash
# Quick status — all DEs
for de in /var/de-agents/*/; do
  de_name=$(basename $de)
  last=$(ls ${de}sessions/ws-${de_name}-*.json 2>/dev/null | sort | tail -1)
  if [ -z "$last" ]; then echo "$de_name: no session"; continue; fi
  python3 -c "
import json
d=json.load(open('$last'))
steps=d.get('steps',[])
actions=sum(1 for s in steps if s.get('type')=='action')
print(f'$de_name: {d[\"status\"]} ({len(steps)} steps, {actions} actions)')
"
done

# pre_fetch smoke test for research DEs
for de in /var/de-agents/grow_*/; do
  de_name=$(basename $de)
  echo -n "$de_name pre_fetch soav: "
  python3 ${de}workspace/measure_soav.py 2>/dev/null || echo "ERROR"
done

# Core service health (adapt to your deployment)
curl -sf http://localhost:8769/health && echo "de-backend OK" || echo "de-backend DOWN"
```

---

## DE Type Classification Template

| DE Name | Type | Step Budget | pre_fetch Notes |
|---|---|---|---|
| ops | monitoring | ≤ 40 | Generic pre_fetch OK |
| shield | security | ≤ 55 | Generic pre_fetch OK |
| flow | coordination | ≤ 25 | Generic pre_fetch OK |
| coach | analysis | ≤ 35 | Generic pre_fetch OK |
| grow_* | research | ≤ 30 | Add run_benchmark_if_stale() |
| geo | research | ≤ 35 | Add run_benchmark_if_stale() |
| content_* | content | ≤ 40 | Generic pre_fetch OK |

---

## Known Issues & Common Fixes

| Symptom | Root Cause | Fix |
|---|---|---|
| Research DE: 43-55 steps, max_iter | Perplexity queries run in session | Add `run_benchmark_if_stale()` to pre_fetch.py |
| Any DE: measure returns `not_measured` | measure script reads wrong path | Update path in measure script or kpis.yaml |
| Any DE: max_iter every session | STOP block too vague or missing | Add/rewrite STOP block with hard limits |
| Monitoring DE: more steps than expected | Too many services, no pre-aggregation | Create run.py script that pre-aggregates all checks |
| Any DE: 65+ steps | No STOP block at all | Add ## STOP section as first content in job.md |

---

*Update this document after every health audit. Every entry goes here — never delete, only amend or mark changed.*
