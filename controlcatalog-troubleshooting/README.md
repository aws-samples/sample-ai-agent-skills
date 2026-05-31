# AWS Control Catalog Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Control Catalog problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Control Catalog — control listing, control details, objective mapping, common controls, regional availability, and API usage.

### Activate When

- Control listing failures or empty results
- Control detail retrieval errors
- Objective mapping issues
- Common control identification problems
- Regional availability questions
- API usage errors
- General Control Catalog errors

---

## Skill Structure

```
controlcatalog-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-list-controls.md
    ├── A2-filter-controls.md
    ├── B1-control-details.md
    ├── B2-control-properties.md
    ├── C1-objective-mapping.md
    ├── C2-objective-listing.md
    ├── D1-common-control-listing.md
    ├── D2-framework-mapping.md
    ├── E1-regional-availability.md
    ├── E2-region-specific-controls.md
    ├── F1-api-errors.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Control Listing** | A1–A2 | List, filter |
| **B — Control Details** | B1–B2 | Details, properties |
| **C — Objectives** | C1–C2 | Mapping, listing |
| **D — Common Controls** | D1–D2 | Listing, framework mapping |
| **E — Regional** | E1–E2 | Availability, region-specific |
| **F — API Usage** | F1 | API errors |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `controlcatalog:*`, `controltower:List*`, `iam:Get*` permissions

---

## License

MIT-0
