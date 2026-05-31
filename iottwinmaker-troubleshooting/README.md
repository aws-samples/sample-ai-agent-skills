# AWS IoT TwinMaker Diagnostics Skill

Agent skill for investigating and troubleshooting AWS IoT TwinMaker problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS IoT TwinMaker — workspaces, entities, components, scenes, data connectors, sync jobs, and Grafana integration.

### Activate When

- Workspace creation or configuration failures
- Entity or component type errors
- Scene loading or rendering issues
- Data connector Lambda failures
- Sync job errors or conflicts
- Grafana plugin configuration problems
- Property value retrieval failures
- General IoT TwinMaker errors

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Workspace** | A1–A2 | Creation failures, configuration |
| **B — Entity/Component** | B1–B3 | Entity errors, component types, properties |
| **C — Scenes** | C1–C2 | Scene loading, 3D rendering |
| **D — Data Connectors** | D1–D2 | Connector failures, Lambda functions |
| **E — Sync Jobs** | E1–E2 | Sync failures, conflict resolution |
| **F — Grafana** | F1–F2 | Plugin setup, dashboard issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with IoTTwinMaker permissions
- Permissions: `iottwinmaker:Get*`, `iottwinmaker:List*`

---

## License

MIT-0
