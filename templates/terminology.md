# Terminology & KPI Definitions
# Organization: [ORG NAME]
# Last updated: [YYYY-MM-DD]
# Owner: [WHO MAINTAINS THIS]

## Purpose
This file is the single source of truth for how this organization defines its terms and measures its KPIs.
Every Digital Employee MUST reference this file when interpreting data. Never invent definitions.

---

## Key Terms

### [Term 1]
**Definition:** How this org understands this term.
**Example:** "A lead is a person who submitted our form AND confirmed via email."
**Not to be confused with:** Related terms that mean something different here.

### [Term 2]
**Definition:** ...

---

## KPI Definitions

### [KPI Name]
| Field | Value |
|-------|-------|
| **What it measures** | |
| **Formula** | e.g., `(Churned in month / Active at start of month) × 100` |
| **Data source** | e.g., Salesforce → Opportunities → CloseDate = last month, Stage = "Churned" |
| **Frequency** | e.g., Monthly, last day of month |
| **Owner** | Which DE is responsible for tracking this |
| **Baseline** | e.g., 4.2% (measured 2026-01) |
| **Target** | e.g., ≤3.5% by Q4 |

### [KPI Name 2]
| Field | Value |
|-------|-------|
| **What it measures** | |
| **Formula** | |
| **Data source** | |
| **Frequency** | |
| **Owner** | |
| **Baseline** | |
| **Target** | |

---

## Data Sources

| System | What it contains | Who can access |
|--------|-----------------|----------------|
| [e.g., Salesforce] | CRM, contacts, donations, opportunities | [API read-only] |
| [e.g., Meta Ads] | Ad performance, spend, audiences | [Marketing manager] |

---

## Decision Authority

| Decision type | Human approval required | Can DE act autonomously |
|---|---|---|
| [e.g., Email campaign launch] | Yes — campaign manager | No |
| [e.g., Segment update] | Yes — data team | No |
| [e.g., Internal report] | No | Yes |

---

## Exclusions & Edge Cases

- [Term]: When X happens, it does NOT count as Y because...
- [Metric]: Exclude [segment] from calculations because...

