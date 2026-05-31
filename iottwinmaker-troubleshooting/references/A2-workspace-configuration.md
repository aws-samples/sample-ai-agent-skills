---
title: "A2 — Workspace Configuration Issues"
description: "Diagnose workspace configuration and access problems"
status: active
severity: MEDIUM
triggers: ["workspace error", "access denied", "workspace configuration"]
owner: devops-agent
objective: "Identify and resolve workspace configuration issues"
context: "Configuration issues occur when: IAM role permissions changed, S3 bucket policy conflicts, or workspace in error state."
---

## Phase 1 — Triage

MUST:
- Check workspace status: `aws iottwinmaker get-workspace --workspace-id <id>`
- Verify IAM role still has required permissions
- Check S3 bucket accessibility
- Review CloudTrail for recent workspace API errors

SHOULD:
- Verify workspace IAM role has permissions for all integrated services
- Check if workspace is in ACTIVE state

MAY:
- List workspace resources (entities, scenes, component types)

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions
- Restore S3 bucket access
- Update workspace if configuration changed

SHOULD:
- Monitor workspace health
- Document required IAM permissions

## Output Format

```yaml
root_cause: "workspace_config — <specific_cause>"
evidence:
  - type: workspace_status
    content: "<workspace details>"
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
