---
title: "Z1 — General TwinMaker Troubleshooting"
description: "Catch-all runbook for unclassified TwinMaker issues"
status: active
severity: MEDIUM
triggers: ["TwinMaker not working", "something wrong with TwinMaker"]
owner: devops-agent
objective: "Systematically triage unclassified TwinMaker issues"
---

## Phase 1 — Triage

MUST:
- List workspaces: `aws iottwinmaker list-workspaces`
- Check workspace status: `aws iottwinmaker get-workspace --workspace-id <id>`
- List entities: `aws iottwinmaker list-entities --workspace-id <id>`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iottwinmaker.amazonaws.com --max-results 20`

SHOULD:
- Check IAM permissions
- Verify S3 bucket accessibility
- Check service health

## Phase 2 — Route to specific runbook

Based on findings, route to A1-A2, B1-B3, C1-C2, D1-D2, E1-E2, or F1-F2.

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: triage_summary
    content: "<initial findings>"
severity: MEDIUM
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete workspaces containing active digital twin entities
- Never modify component types that are referenced by production entities
- Never remove data connectors without confirming alternative data sources
- Never alter scene configurations used by active monitoring dashboards
- Never delete entity hierarchies without understanding downstream visualization impact

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous component type definitions if entity data binding breaks
- Revert scene configuration changes if 3D visualization stops rendering
- Re-enable previous data connector settings if twin data synchronization fails

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
