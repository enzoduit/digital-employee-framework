# KPI System — Digital Employee Framework

## Overview

Every DE has a `workspace/kpis.yaml` that defines its measurable KPIs. The generic `pre_fetch.py` reads this file, runs each measurement command, and emits a structured briefing before the ReAct loop starts — without any LLM call.

```
session starts
     │
     ▼
pre_fetch.py  ← reads kpis.yaml, runs measure commands (10s timeout each)
     │           writes metrics.json with new values + history
     ▼
briefing printed to stdout (injected as trigger_context)
     │
     ▼
ReAct loop begins with KPI state visible
```

## kpis.yaml Schema

```yaml
de: "<de_name>"           # must match directory name in /var/de-agents/
kpis:
  - id: <snake_case>      # unique identifier, also used as key in metrics.json
    name: <Human Name>    # shown in briefing
    target: <number>      # numeric target value
    unit: "%"             # display unit (%, leads, USD, pages, ...)
    direction: up         # up = higher is better, down = lower is better
    measure: "<command>"  # shell command run from DE_DIR, must print one value to stdout
    frequency: weekly     # daily | weekly | per_session (informational only, not enforced yet)
```

## Measurement Commands

The `measure` command runs with CWD = `/var/de-agents/<de>/` and a 10-second timeout.
It must print exactly one value to stdout:
- A number (int or float) → stored and compared against target
- `not_measured` → shown as ⚠ in briefing, not stored
- Empty / error → treated as not_measured

## Available measure_*.py Scripts

### `measure_soav.py`
Reads cached SoAV (Share of AI Voice) score. Deployed to all grow_* and geo DEs.

```
Reads: workspace/soav_latest.json → .soav_score or .soav_pct
       workspace/benchmark_*.json → .soav_pct (fallback, e.g. grow_agentic_living)
Outputs: 0.0–100.0 (percentage) or 'not_measured'
```

To populate the cache, the DE's benchmark script must write:
```json
// workspace/soav_latest.json
{"soav_score": 34.5, "timestamp": "2026-09-22T10:00:00Z"}
```

### `measure_sessions.py`
Scans all session files across all DEs for the last 7 days. Deployed to flow and max.

```
Usage: python3 workspace/measure_sessions.py --metric <metric>
Metrics:
  timed_out_rate  → % sessions with status=max_iterations_reached
  no_output_rate  → % sessions with no summary field
  total_sessions  → raw count
```

### `measure_cost.py`
Estimates cost from session step counts. Deployed to max.

```
Usage: python3 workspace/measure_cost.py --trigger <cron|all> --metric <metric>
Metrics:
  avg_cost    → average USD cost per matching session (default)
  waste_rate  → % sessions with no output
  total_cost  → total USD for period
Cost model: haiku=0.0003/step, sonnet=0.003/step, default=0.001/step
```

## metrics.json Format

Written to `/var/de-agents/<de>/metrics.json` after each pre_fetch run:

```json
{
  "last_updated": "2026-09-22T06:20:00Z",
  "kpis": [
    {
      "id": "soav_score",
      "name": "SoAV Score (Perplexity)",
      "target": 30,
      "unit": "%",
      "direction": "up",
      "value": 34.5,
      "updated": "2026-09-22T06:20:00Z",
      "history": [
        {"ts": "2026-09-15", "value": 28.0}
      ]
    }
  ]
}
```

`history` stores the last 10 previous values (only written on change).

## Briefing Format

```
=== GROW_MINIMIST BRIEFING — 2026-09-22 ===

OFF TRACK — action needed:
  ✗ Leads (charity retail directors): 1 leads (target: 3 leads)

Not yet measured:
  ⚠ SoAV Score (Perplexity): not measured (target: 30 %)

On track:
  ✓ Some KPI: 42 % (target: 30 %)

Active experiments:
  ## Experiment: ...

RULE: Address off-track KPIs first. One focused action per session.
```

## KPI Definitions by DE

| DE | KPI | Target | Source |
|----|-----|--------|--------|
| <de-name> | SoAV Score | 30% | soav_latest.json |
| <de-name> | Leads | 3 | workspace/leads.json |
| grow_agentfabric | SoAV Score | 25% | soav_latest.json |
| grow_agentic_living | SoAV Score | 30% | benchmark_*.json |
| growed | SoAV Score (personal brand) | 20% | soav_latest.json |
| grow_engelreal | SoAV Score | 30% | soav_latest.json |
| grow_flyraising | SoAV Score | 25% | soav_latest.json |
| grow_rflect | SoAV Score | 50% | soav_latest.json |
| grow_studyond | SoAV Score | 25% | soav_latest.json |
| grow_vwupass | SoAV Score | 30% | soav_latest.json |
| geo | SoAV Score | 30% | soav_latest.json |
| geo | Active satellite pages | 10 | workspace/pages.json |
| flow | Timed-out rate (7d) | ≤10% | session files |
| flow | No-output rate (7d) | ≤15% | session files |
| max | Avg cost per cron session | ≤$0.05 | session files |
| max | Waste rate (no output) | ≤20% | session files |
| coach | Training adherence | 90% | workspace/metrics_cache.json |
| scribe | Pending experiment docs | 0 | metrics.json |
| shield | Core services healthy | 100% | systemctl |
| growth | Weekly visitors | 100 | workspace/traffic_cache.json |

## Benchmark Mode

Use Benchmark Mode when a client has **no historical data and no agreed numeric target**. Instead of guessing a target upfront, the first measurement becomes the baseline, and the target is auto-calculated as a 10% improvement from there.

### kpis.yaml with Benchmark Mode

```yaml
de: "<de_name>"
kpis:
  - id: churn_rate
    name: "Monthly Churn Rate"
    description: "% Donors who cancel in a given month"
    measure: "python3 workspace/measure_churn.py"
    unit: "%"
    direction: "lower_is_better"  # use 'up' for metrics where higher = better
    target: null                  # null = not set yet; will be auto-filled after first run
    target_auto: true             # enables benchmark mode
    target_improvement: 0.10     # 10% improvement from baseline
    frequency: monthly
```

### How `pre_fetch.py` handles benchmark mode

On every run, after measuring each KPI:

```python
# Pseudocode — actual logic in framework/pre_fetch.py
for kpi in kpis:
    value = measure_kpi(kpi)                    # run measure command
    target = kpi.get('target')
    target_auto = kpi.get('target_auto', False)

    if value is not None and target is None and target_auto:
        # First real measurement → set baseline as target
        direction = kpi.get('direction', 'up')
        improvement = float(kpi.get('target_improvement', 0.10))

        if direction == 'up':
            new_target = round(float(value) * (1 + improvement), 4)
        else:  # lower_is_better
            new_target = round(float(value) * (1 - improvement), 4)

        # Write concrete target back into kpis.yaml
        update_kpis_yaml_target(kpi['id'], new_target)  # sets target: <value>
        # Mark in metrics.json so we know this was auto-set
        kpi['target'] = new_target
        kpi['target_auto_set'] = True
        kpi['baseline'] = float(value)
```

**Result:** After the first session, `kpis.yaml` has a real `target:` value. All subsequent runs use normal on-track/off-track comparison.

### Direction rules for auto-target calculation

| `direction` value | Formula | Example |
|------------------|---------|--------|
| `up` | `baseline × 1.10` | SoAV: 30% → target 33% |
| `lower_is_better` | `baseline × 0.90` | Churn: 8% → target 7.2% |

Customize the multiplier via `target_improvement` (0.10 = 10%, 0.20 = 20%, etc.).

### What the briefing shows in benchmark mode

Before first measurement:
```
⚠ Monthly Churn Rate: not measured yet (benchmark mode — first run sets baseline)
```

After first measurement (baseline set, shown next run):
```
✓ Monthly Churn Rate: 8.2 % (target: 7.4 % — auto-set from baseline 8.2%)
```

Off track:
```
✗ Monthly Churn Rate: 9.1 % (target: 7.4 % — auto-set from baseline 8.2%)
```

---

## Adding KPIs for a New DE

1. Create `/var/de-agents/<de>/workspace/kpis.yaml` following the schema above.
2. Copy `pre_fetch.py` from `de-framework/framework/` into the workspace.
3. If you need SoAV tracking, copy `measure_soav.py` too.
4. Test: `python3 /var/de-agents/<de>/workspace/pre_fetch.py`
5. Add an entry to this doc's table.

## Data Gaps (as of 2026-09-22)

These KPIs will show `not_measured` until the cache files are created:

| DE | KPI | Missing file | How to fix |
|----|-----|-------------|------------|
| All grow_* (most) | SoAV Score | workspace/soav_latest.json | Run Perplexity benchmark → write soav_latest.json |
| <de-name> | Leads | workspace/leads.json | Create leads tracking → `{"leads": [...]}` |
| geo | Satellite pages | workspace/pages.json | Create pages tracking → `{"pages": [...]}` |
| coach | Training adherence | workspace/metrics_cache.json | Garmin sync → write metrics_cache.json |
| growth | Weekly visitors | workspace/traffic_cache.json | Analytics integration → write traffic_cache.json |

## Files

```
de-framework/framework/
├── pre_fetch.py          ← generic, deploy to all DE workspaces
├── measure_soav.py       ← for grow_*, geo
├── measure_sessions.py   ← for flow, max
├── measure_cost.py       ← for max
└── kpis/
    ├── <de-name>.yaml
    ├── grow_agentfabric.yaml
    ├── grow_agentic_living.yaml
    ├── growed.yaml
    ├── grow_engelreal.yaml
    ├── grow_flyraising.yaml
    ├── grow_rflect.yaml
    ├── grow_studyond.yaml
    ├── grow_vwupass.yaml
    ├── geo.yaml
    ├── flow.yaml
    ├── max.yaml
    ├── coach.yaml
    ├── scribe.yaml
    ├── shield.yaml
    └── growth.yaml
```

---

## Lessons Learned — From Real Production Runs

*Collected 2026-09-26 after first full quarter of operation.*

### Lesson 1: Monitoring DEs — target ≤ 40 steps (achievable)

Monitoring DEs (ops-style) that check 3-7 services typically run in 25-40 steps on a complete, successful session. This is the most reliable DE type. Key success factors:
- One `run.py` script pre-aggregates all service checks
- STOP block references specific log file, not "take one action"
- Session is `complete`, not `max_iterations_reached`

### Lesson 2: Research DEs — target ≤ 30 steps (requires pre_fetch benchmark cache)

GEO/research DEs (grow_*-style) that benchmark AI search visibility WILL exceed 43 steps if they run Perplexity queries inside the ReAct loop. Each query = 1-3 tool calls. 12 queries × 3 avg = 36 steps before any content writing.

**The fix:** move benchmark queries to `pre_fetch.py` via `run_benchmark_if_stale()`. Session only reads the cached result and writes ONE content improvement.

```python
# pre_fetch.py addition for research DEs
def run_benchmark_if_stale():
    cache_age = get_soav_cache_age()  # Check soav_history.json or benchmark_*.json
    if cache_age > 21600:  # 6 hours
        subprocess.run(['python3', str(benchmark_script)], timeout=300)
```

### Lesson 3: STOP block must be SPECIFIC — "stop after writing X to Y file"

Generic STOP blocks fail. These patterns do NOT work:
- ❌ "stop after 5-10 steps" — agent decides the cron work requires more
- ❌ "identify ONE action" — agent spends 20 steps identifying before acting

These patterns work:
- ✅ "DONE = workspace/log.md updated. Stop immediately after that write."
- ✅ "HARD LIMIT: max 4 tool calls. After 4: write whatever you have and stop."
- ✅ "DO NOT run Perplexity in-session. DO NOT search for API keys."

### Lesson 4: measure_soav.py must know real data paths

The generic `measure_soav.py` reads from `workspace/soav_latest.json`. But many agents store benchmark results in a different path (e.g., `/root/.openclaw/workspace/agents/{de}/soav_history.json`). When the path is wrong, `measure_soav.py` returns `not_measured` — which triggers the agent to run the full benchmark in-session.

**The fix:** Add path fallbacks to `measure_soav.py`:
```python
# Fallback: agent workspace soav_history.json
agent_workspace = Path('/root/.openclaw/workspace/agents') / DE_DIR.name
soav_hist = agent_workspace / 'soav_history.json'
if soav_hist.exists():
    # Extract score from last cycle ...
```

### Lesson 5: No STOP block = guaranteed max_iterations

One DE (aria) had NO STOP block at all. It consistently hit 65 steps. Adding a STOP block with concrete done-conditions dropped it to the target range immediately.

Every DE job.md must have a `## STOP` section as the FIRST content block.

