---
title: "A2 — RDS Oracle Instance Class Issues"
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
  - "vCPU limit"
owner: devops-agent
objective: "Resolve instance class compatibility, sizing, and modification issues for RDS Oracle"
context: "Oracle SE2 is limited to 16 vCPUs. Instance class changes require downtime. Not all instance classes support all Oracle versions. BYOL licensing ties to specific CPU counts."
---

## Phase 1 — Triage

MUST:
- Check current instance class: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].DBInstanceClass'`
- Check Oracle edition: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].Engine'`
- List valid instance classes for the engine: `aws rds describe-orderable-db-instance-options --engine oracle-ee --engine-version <version> --query 'OrderableDBInstanceOptions[].{Class:DBInstanceClass,vCPU:Vcpu,Memory:MemoryMiB}'`
- Check pending modifications: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].PendingModifiedValues'`

SHOULD:
- Check SE2 vCPU limit (max 16 vCPUs): verify target class doesn't exceed limit
- Review CloudWatch metrics to validate sizing need:
  - `CPUUtilization` — sustained > 80% suggests scale up
  - `FreeableMemory` — consistently low suggests memory-optimized class
  - `DatabaseConnections` — near limit suggests larger class
- Check if Multi-AZ — modification applies to both primary and standby

MAY:
- Compare current vs target instance class pricing
- Check Enhanced Monitoring for detailed CPU/memory breakdown

## Phase 2 — Remediate

MUST:
- For SE2 vCPU limit: choose an instance class with ≤ 16 vCPUs or migrate to EE
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
- Update BYOL license counts if changing CPU count
- Monitor after resize for memory-related ORA errors (ORA-04031)

MAY:
- Use Multi-AZ failover to minimize downtime during resize

## Common Issues

- symptoms: "Cannot modify to db.r5.4xlarge on oracle-se2"
  diagnosis: "SE2 limited to 16 vCPUs. db.r5.4xlarge has 16 vCPUs (OK), but db.r5.8xlarge (32 vCPUs) would fail."
  resolution: "Choose an instance class within the SE2 vCPU limit."

- symptoms: "Modification stuck in pending"
  diagnosis: "Modification scheduled for next maintenance window."
  resolution: "Apply immediately or wait for maintenance window."

- symptoms: "ORA-04031 after resize to smaller instance"
  diagnosis: "SGA too large for new instance memory. Parameter group formulas reference DBInstanceClassMemory."
  resolution: "Adjust sga_target and pga_aggregate_target for the new instance class memory."

## Safety Ratings
- GREEN: describe-db-instances, describe-orderable-db-instance-options, CloudWatch CPUUtilization/FreeableMemory/DatabaseConnections metrics — read-only inspection
- YELLOW: modify-db-instance (instance class change) — recoverable but requires downtime
- RED: delete-db-instance, force-failover in production — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Instance class change requires downtime (typically 10-30 minutes)"
- "Fix requires failover of Multi-AZ instance"
- "SE2 vCPU limit requires migration to Enterprise Edition"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, BYOL license counts
- MEDIUM: instance class details, CloudWatch metrics, pending modifications

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix sizing issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"

## Phase 3 — Rollback
- "Restore from snapshot if instance class change causes ORA-04031 or other memory errors"
- "Revert parameter group changes (sga_target, pga_aggregate_target) and reboot if needed"
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
