---
name: iottwinmaker-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS IoT TwinMaker problems
  by analyzing workspaces, entities, components, scenes, data connectors,
  sync jobs, and Grafana integration using structured runbooks. Activate
  when: workspace creation failures, entity/component errors, scene loading
  issues, data connector failures, sync job problems, Grafana plugin issues,
  or the user says something is wrong with IoT TwinMaker.
compatibility: >
  Requires AWS CLI or SDK access with IoTTwinMaker, IAM, S3, CloudWatch,
  and Grafana permissions as applicable.
---

# AWS IoT TwinMaker Diagnostics

## When to use

Any AWS IoT TwinMaker investigation — workspaces, entities, components, scenes, data connectors, sync jobs, or Grafana integration troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws iottwinmaker list-workspaces
aws iottwinmaker get-workspace --workspace-id <id>
aws iottwinmaker list-entities --workspace-id <id>
aws iottwinmaker get-entity --workspace-id <id> --entity-id <entity-id>
```

### Step 2 — Domain deep dive

```
aws iottwinmaker list-component-types --workspace-id <id>
aws iottwinmaker get-component-type --workspace-id <id> --component-type-id <type-id>
aws iottwinmaker list-scenes --workspace-id <id>
aws iottwinmaker get-scene --workspace-id <id> --scene-id <scene-id>
```

### Step 3 — Detailed investigation

```
aws iottwinmaker list-sync-jobs --workspace-id <id>
aws iottwinmaker get-sync-job --workspace-id <id> --sync-source <source>
aws iottwinmaker get-property-value --workspace-id <id> --entity-id <entity-id> --component-name <name> --selected-properties '["<prop>"]'
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iottwinmaker.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any IoT TwinMaker issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-workspace` | Check workspace status and S3 bucket |
| `get-entity` | Check entity and component configuration |
| `get-component-type` | Check component type definition |
| `get-scene` | Check scene configuration |
| `get-sync-job` | Check sync job status |
| `get-property-value` | Get current property values |

## Anti-hallucination rules

1. Always cite specific workspace IDs, entity IDs, or API responses as evidence.
2. Workspaces require an S3 bucket for content storage. Never assume automatic storage.
3. Component types define the schema; components are instances on entities.
4. Data connectors use Lambda functions. Never assume direct database access.
5. Scenes are 3D visualizations stored in S3. Never confuse with dashboards.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Workspace | A1-A2 | Creation failures, configuration |
| B — Entity/Component | B1-B3 | Entity errors, component types, properties |
| C — Scenes | C1-C2 | Scene loading, 3D rendering |
| D — Data Connectors | D1-D2 | Connector failures, Lambda functions |
| E — Sync Jobs | E1-E2 | Sync failures, conflict resolution |
| F — Grafana | F1-F2 | Plugin setup, dashboard issues |
| Z — Catch-All | Z1 | General troubleshooting |
