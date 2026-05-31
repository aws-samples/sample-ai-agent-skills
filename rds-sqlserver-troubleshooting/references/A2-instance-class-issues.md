---
title: "A2 — RDS SQL Server Instance Class Issues"
description: "Diagnose instance class sizing, compatibility, and modification problems"
status: active
severity: MEDIUM
triggers:
  - "instance class"
  - "instance type"
  - "resize"
  - "modify instance"
  - "db.m5"
  - "db.r5"
  - "Express edition limit"
owner: devops-agent
objective: "Resolve instance class compatibility, sizing, and modification issues for RDS SQL Server"
context: "SQL Server Express is limited to specific small instance classes. Instance class changes require downtime. Not all instance classes support all SQL Server editions or versions. License Included pricing varies by edition and instance class."
---

## Phase 1 — Triage

MUST:
- Check current instance class: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].DBInstanceClass'`
- Check SQL Server edition: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].Engine'`
- List valid instance classes for the engine: `aws rds describe-orderable-db-instance-options --engine sqlserver-se --engine-version <version> --query 'OrderableDBInstanceOptions[].{Class:DBInstanceClass,vCPU:Vcpu,Memory:MemoryMiB}'`
- Check pending modifications: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].PendingModifiedValues'`

SHOULD:
- Review CloudWatch metrics to validate sizing need:
  - `CPUUtilization` — sustained > 80% suggests scale up
  - `FreeableMemory` — consistently low suggests memory-optimized class
  - `DatabaseConnections` — near limit suggests larger class
- Check if Multi-AZ — modification applies to both primary and secondary
- Verify Express edition limits (10 GB database size, 1 GB buffer pool)

MAY:
- Compare current vs target instance class pricing
- Check Enhanced Monitoring for detailed CPU/memory breakdown

## Phase 2 — Remediate

MUST:
- For Express edition limits: migrate to Web or Standard edition for larger instances
- For modification failures: apply during maintenance window or with `--apply-immediately`
- Verify parameter group compatibility with new instance class (memory-based formulas)

```
aws rds modify-db-instance \
  --db-instance-identifier <instance-id> \
  --db-instance-class <new-class> \
  --apply-immediately
```

SHOULD:
- Plan for downtime during instance class changes (typically 10-30 minutes)
- Monitor after resize for memory pressure issues
- Multi-AZ instances failover during modification to reduce downtime

MAY:
- Use Multi-AZ failover to minimize downtime during resize

## Common Issues

- symptoms: "Cannot modify to db.r5.4xlarge on sqlserver-ex"
  diagnosis: "Express edition supports only limited instance classes."
  resolution: "Migrate to Web or Standard edition for larger instance classes."

- symptoms: "Modification stuck in pending"
  diagnosis: "Modification scheduled for next maintenance window."
  resolution: "Apply immediately or wait for maintenance window."

- symptoms: "Performance degradation after resize to smaller instance"
  diagnosis: "Buffer pool reduced. Max server memory auto-adjusted by RDS."
  resolution: "Monitor buffer pool hit ratio. Scale back up if needed."

## Safety Ratings
- GREEN: describe-db-instances, describe-orderable-db-instance-options, CloudWatch CPUUtilization/FreeableMemory/DatabaseConnections metrics — read-only inspection
- YELLOW: modify-db-instance (instance class change) — recoverable but requires downtime
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Instance class change requires downtime (typically 10-30 minutes)"
- "Fix requires failover of Multi-AZ instance"
- "Express edition limits require migration to higher edition"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, license details
- MEDIUM: instance class details, CloudWatch metrics, pending modifications

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix sizing issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest granting sysadmin role"

## Phase 3 — Rollback
- "Restore from snapshot if instance class change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "instance_class — <specific_cause>"
evidence:
  - type: instance_config
    content: "<current instance class and edition>"
  - type: metrics
    content: "<CloudWatch metrics summary>"
severity: MEDIUM
mitigation:
  immediate: "Modify to appropriate instance class"
  long_term: "Implement CloudWatch alarms for proactive scaling"
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
