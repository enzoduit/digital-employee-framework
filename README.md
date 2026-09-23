# Digital Employee Framework

A framework for building autonomous Digital Employees powered by OpenClaw.

## What is a Digital Employee?

A Digital Employee (DE) is an AI agent that owns a single metric, works autonomously toward a defined goal, and escalates to humans only when necessary.

Each DE has:
- A **mission** — one sentence, owns one metric
- **KPIs** — measured automatically before every session
- **Authority levels** — what it can do alone vs. what needs human approval
- A **workspace** — persistent storage for experiments and results
- A **schedule** — cron-driven, not reactive

## Quick Start

→ Read [QUICKSTART.md](QUICKSTART.md)

## Key Concepts

| Concept | File |
|---------|------|
| How to set up a client | `QUICKSTART.md` |
| KPI schema + benchmark mode | `docs/kpi-system.md` |
| Available tools in a DE session | `docs/runtime-environment.md` |
| Integration discovery | `docs/integration-discovery.md` |
| Chief agent pattern | `docs/patterns/chief-agent.md` |
| Transversal agent pattern | `docs/patterns/transversal-agent.md` |
| pre_fetch.py pattern | `docs/pre-fetch-pattern.md` |

## Repository Structure

```
digital-employee-framework/
├── QUICKSTART.md              ← Start here if you are a setup agent
├── README.md                  ← This file
├── examples/
│   └── job-template.md        ← Base template for any DE's job.md
├── docs/
│   ├── kpi-system.md          ← KPI schema, benchmark mode
│   ├── runtime-environment.md ← Tools available in a DE session
│   ├── integration-discovery.md ← How to discover and configure integrations
│   ├── pre-fetch-pattern.md   ← How pre_fetch.py works
│   └── patterns/
│       ├── chief-agent.md     ← Coordinating agent pattern
│       └── transversal-agent.md ← Shared/cross-area agent pattern
├── framework/
│   └── pre_fetch.py           ← Generic KPI measurement script
└── scripts/
    ├── new-client-setup.sh    ← Bootstrap new client directories
    └── discover-integrations.sh ← Probe available integrations
```

## License

MIT
