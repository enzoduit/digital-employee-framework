# Terminology File Pattern

**Every DE deployment MUST include a `terminology.md` in the shared workspace.**

## Why

Without a shared terminology file:
- DE A defines "lead" one way, DE B defines it differently
- KPI calculations are inconsistent across DEs
- When the org changes a definition, it has to be updated in 10 places
- New DEs learn wrong definitions from hallucination

With a shared terminology file:
- Single source of truth
- DEs reference it explicitly: "According to terminology.md, a lead is..."
- Changes propagate automatically to all DEs that reference it
- Auditable — you can see when a definition changed and why

## Where It Lives

```
/var/de-agents/<client>/shared/terminology.md
```

All DEs in the same client deployment share one file. Not per-DE — per-organization.

## How DEs Use It

pre_fetch.py loads it before every session:
```python
terminology = load_file('shared/terminology.md')
briefing += f"\n\n## Organizational Terminology\n{terminology}"
```

DEs are instructed in job.md:
```
When interpreting any metric or term, always check shared/terminology.md first.
Never invent definitions. If a term is not in terminology.md, flag it as undefined
and ask for clarification rather than assuming.
```

## How to Create One

1. Copy `templates/terminology.md` to `/var/de-agents/<client>/shared/terminology.md`
2. Fill in all key terms the client uses
3. For each KPI: formula, data source, frequency, baseline, target
4. Review with client — they know their definitions best
5. DEs will reference it from session 1

## Keeping It Current

- When org changes a definition → update this file, not individual job.md files
- Add a changelog section at the bottom
- DE Coach (or equivalent) is responsible for maintaining it
- Any DE that encounters an undefined term should flag it as a decision for the human

## Template

→ `templates/terminology.md`
