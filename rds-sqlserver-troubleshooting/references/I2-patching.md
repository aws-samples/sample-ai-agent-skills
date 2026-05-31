---
title: "I2 — Patching"
description: "Diagnose RDS SQL Server patching and maintenance window issues"
status: active
severity: MEDIUM
triggers:
  - "patching"
  - "maintenance window"
  - "pending maintenance"
  - "auto minor version upgrade"
  - "cumulative update"
  - "CU"
  - "patch"
owner: devops-agent
objective: "Resolve patching and maintenance window issues for RDS SQL Server"
context: "RDS applies SQL Server Cumulative Updates (CUs) as engine version updates. Patching occurs during the maintenance window unless applied immediately. Auto minor version upgrade can be enabled. Patching causes a brief outage (reboot). Multi-AZ reduces downtime via failover-based patching."
---

## Phase 1 — Triage

MUST:
- Check pending maintenance: `aws rds describe-pending-maintenance-actions --resource-identifier <instance-arn>`
- Check maintenance window: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].PreferredMaintenanceWindow'`
- Check auto minor version upgrade setting: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].AutoMinorVersionUpgrade'`
- Check current version: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].EngineVersion'`

SHOULD:
- Check available engine versions: `aws rds describe-db-engine-versions --engine sqlserver-se --query 'DBEngineVersions[].EngineVersion'`
- Check RDS events for maintenance actions: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --event-categories maintenance --duration 10080`
- Verify Multi-AZ status (reduces patching downtime): `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].MultiAZ'`

MAY:
- Check AWS RDS maintenance schedule announcements
- Review SQL Server CU release notes for known issues

## Phase 2 — Remediate

MUST:
- Apply pending maintenance immediately if needed:
  ```
  aws rds apply-pending-maintenance-action \
    --resource-identifier <instance-arn> \
    --apply-action system-update \
    --opt-in-type immediate
  ```
- Or defer to next maintenance window:
  ```
  aws rds apply-pending-maintenance-action \
    --resource-identifier <instance-arn> \
    --apply-action system-update \
    --opt-in-type next-maintenance
  ```
- Adjust maintenance window to low-traffic period:
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --preferred-maintenance-window "sun:03:00-sun:04:00"
  ```

SHOULD:
- Enable auto minor version upgrade for non-production:
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --auto-minor-version-upgrade
  ```
- Test patches in non-production before applying to production
- Create snapshot before patching

MAY:
- Use blue-green deployment for zero-downtime patching
- Set up EventBridge notifications for maintenance events

## Common Issues

- symptoms: "Unexpected downtime during maintenance window"
  diagnosis: "Pending maintenance applied during the window. Auto minor version upgrade enabled."
  resolution: "Review pending maintenance. Adjust maintenance window. Disable auto upgrade if needed."

- symptoms: "Cannot defer maintenance — required update"
  diagnosis: "Security patches or EOL updates may be mandatory with a deadline."
  resolution: "Plan for the required update. Create snapshot. Apply during low-traffic period."

- symptoms: "Patching takes longer than expected"
  diagnosis: "Major patches or parameter group changes require longer downtime."
  resolution: "Use Multi-AZ to reduce downtime. Plan for extended maintenance window."

## Safety Ratings
- GREEN: describe-pending-maintenance-actions, describe-db-instances, describe-db-engine-versions, describe-events — read-only inspection
- YELLOW: apply-pending-maintenance-action, modify-db-instance (maintenance window) — recoverable but causes brief downtime
- RED: delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Mandatory security patch with deadline"
- "Patching requires extended downtime"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings
- MEDIUM: pending maintenance details, maintenance window configuration, engine version

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix patching issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest skipping mandatory security patches"

## Phase 3 — Rollback
- "Create snapshot before patching to enable rollback"
- "Restore from pre-patch snapshot if patching causes application issues"

## Output Format

```yaml
root_cause: "patching — <specific_cause>"
evidence:
  - type: pending_maintenance
    content: "<pending maintenance actions>"
  - type: maintenance_window
    content: "<current maintenance window>"
severity: MEDIUM
mitigation:
  immediate: "Apply or defer maintenance as appropriate"
  long_term: "Implement patching strategy with testing pipeline"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making databases publicly accessible"
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting automated backups"
