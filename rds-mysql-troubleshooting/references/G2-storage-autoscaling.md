---
title: "G2 — Storage Autoscaling"
description: "Diagnose storage autoscaling failures and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "storage full"
  - "autoscaling"
  - "FreeStorageSpace"
  - "storage-full state"
---

## Phase 1 — Triage

MUST:
- Check free storage: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace --dimensions Name=DBInstanceIdentifier,Value=<id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`
- Check autoscaling config: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].MaxAllocatedStorage'`
- Check storage events: `aws rds describe-events --source-identifier <id> --source-type db-instance --event-categories storage --duration 1440`

SHOULD:
- Verify autoscaling is enabled: `MaxAllocatedStorage` must be > `AllocatedStorage`
- Check if 6-hour cooldown is blocking scaling
- Check if storage-full state was reached before autoscaling triggered

## Phase 2 — Remediate

- Enable autoscaling: `aws rds modify-db-instance --db-instance-identifier <id> --max-allocated-storage 1000 --apply-immediately`
- Autoscaling triggers: free space < 10% AND low space lasts > 5 minutes AND 6 hours since last scaling
- Storage only grows, never shrinks — plan max-allocated-storage carefully
- If storage-full: manually increase storage `--allocated-storage <new-size>`
- Reduce storage usage: purge old data, optimize tables, check binary log retention

## Safety Ratings
- GREEN: describe-db-instances (MaxAllocatedStorage), CloudWatch FreeStorageSpace metrics — read-only inspection
- YELLOW: modify-db-instance (max-allocated-storage, allocated-storage) — recoverable changes
- RED: delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Storage full before autoscaling triggered"
- "Fix requires manual storage increase (6-hour cooldown)"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings
- MEDIUM: storage configuration, autoscaling settings, CloudWatch metrics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix storage issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "Storage increases cannot be reversed — plan increases with sufficient headroom"
- "Wait 6 hours between storage modifications (cooldown period)"
- "Restore from snapshot if storage configuration change causes issues"

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
