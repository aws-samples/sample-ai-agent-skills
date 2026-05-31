# AWS IoT SiteWise Diagnostics Skill

Agent skill for investigating and troubleshooting AWS IoT SiteWise problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS IoT SiteWise — asset models, measurements, transforms, metrics, portals, gateways, data ingestion, property values, and industrial data collection.

### Activate When

- Asset model creation or update failures
- Measurement data not appearing
- Transform computation errors
- Metric aggregation issues
- Portal access or dashboard problems
- Gateway connectivity failures
- Data ingestion errors or throttling
- Property value discrepancies
- Hierarchy configuration issues
- General IoT SiteWise errors

---

## Skill Structure

```
iotsitewise-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-model-creation.md
    ├── A2-hierarchy-issues.md
    ├── B1-measurement-ingestion.md
    ├── B2-transform-errors.md
    ├── B3-metric-aggregation.md
    ├── C1-portal-access.md
    ├── C2-dashboard-issues.md
    ├── D1-gateway-connectivity.md
    ├── D2-data-source-config.md
    ├── E1-data-ingestion-errors.md
    ├── E2-batch-operations.md
    ├── F1-alarm-configuration.md
    ├── F2-notification-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Asset Models** | A1–A2 | Model creation, hierarchy issues |
| **B — Properties** | B1–B3 | Measurements, transforms, metrics |
| **C — Portals** | C1–C2 | Portal access, dashboard issues |
| **D — Gateways** | D1–D2 | Gateway connectivity, data source config |
| **E — Ingestion** | E1–E2 | Data ingestion, batch operations |
| **F — Integration** | F1–F2 | Alarms, notifications |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering asset model vs asset distinction, property type semantics, transform formula validation, metric time window behavior, gateway Greengrass requirements, portal SSO authentication, data ingestion rate limits, timestamp requirements, hierarchy depth limits, property data type matching, batch operation limits, and alarm state machine behavior.

---

## Prerequisites

- AWS CLI v2 with IoTSiteWise permissions
- Permissions: `iotsitewise:Describe*`, `iotsitewise:List*`, `iotsitewise:Get*`, `iotsitewise:BatchGet*`

---

## License

MIT-0
