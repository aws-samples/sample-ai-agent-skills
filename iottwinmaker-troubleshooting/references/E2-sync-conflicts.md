---
title: "E2 — Sync Conflict Resolution"
description: "Diagnose sync job conflict and merge issues"
status: active
severity: MEDIUM
triggers: ["sync conflict", "entity overwritten", "merge error"]
owner: devops-agent
objective: "Identify and resolve sync conflict issues"
context: "Conflicts occur when: manual entity changes conflict with sync source, multiple sync sources overlap, or entity deleted during sync."
---

## Phase 1 — Triage

MUST:
- Check sync job error details for conflict information
- Identify conflicting entities
- Determine if manual changes were overwritten
- Check sync source for expected entity state

SHOULD:
- Review sync job history for patterns
- Check if multiple sync sources target same entities
- Verify conflict resolution strategy

## Phase 2 — Remediate

MUST:
- Resolve conflicts by choosing authoritative source
- Re-run sync after fixing conflicts
- Avoid manual changes to synced entities

SHOULD:
- Establish clear ownership (sync vs manual) per entity
- Document sync conflict resolution procedures

## Output Format

```yaml
root_cause: "sync_conflict — <specific_cause>"
evidence:
  - type: conflict_details
    content: "<conflicting entities>"
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
