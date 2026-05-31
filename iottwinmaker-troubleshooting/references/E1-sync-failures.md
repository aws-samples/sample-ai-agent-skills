---
title: "E1 — Sync Job Failures"
description: "Diagnose sync job execution failures"
status: active
severity: HIGH
triggers: ["sync failed", "sync error", "sync not running"]
owner: devops-agent
objective: "Identify and resolve sync job failures"
context: "Sync failures occur when: source unavailable, permissions missing, entity conflicts, or sync configuration invalid."
---

## Phase 1 — Triage

MUST:
- Check sync job status: `aws iottwinmaker get-sync-job --workspace-id <id> --sync-source <source>`
- List sync jobs: `aws iottwinmaker list-sync-jobs --workspace-id <id>`
- Verify sync source is accessible (e.g., SiteWise)
- Check workspace IAM role has permissions for sync source

SHOULD:
- Check for entity conflicts between sync and manual changes
- Review sync job error messages
- Verify sync source configuration

## Phase 2 — Remediate

MUST:
- Fix sync source accessibility
- Grant workspace role permissions for sync source
- Resolve entity conflicts

SHOULD:
- Avoid manual entity changes that conflict with sync
- Monitor sync job health

## Output Format

```yaml
root_cause: "sync_failure — <specific_cause>"
evidence:
  - type: sync_status
    content: "<sync job details>"
severity: HIGH
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
