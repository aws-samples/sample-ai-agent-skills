---
title: "A1 — RDS MySQL Launch Failures"
description: "Diagnose failures when creating or launching an RDS MySQL instance"
status: active
severity: HIGH
triggers:
  - "launch failed"
  - "create db instance failed"
  - "InsufficientDBInstanceCapacity"
  - "cannot create"
---

## Phase 1 — Triage

MUST:
- Check RDS events: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`
- Verify engine version availability: `aws rds describe-db-engine-versions --engine mysql --query 'DBEngineVersions[].EngineVersion'`
- Check instance class availability: `aws rds describe-orderable-db-instance-options --engine mysql --engine-version <version> --query 'OrderableDBInstanceOptions[].DBInstanceClass'`
- Verify subnet group spans 2+ AZs: `aws rds describe-db-subnet-groups --db-subnet-group-name <subnet-group>`
- Confirm parameter group family matches engine: `aws rds describe-db-parameter-groups --db-parameter-group-name <param-group>`

SHOULD:
- Verify security group exists: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check KMS key if encrypted: `aws kms describe-key --key-id <key-id>`
- Check service quotas: `aws service-quotas get-service-quota --service-code rds --quota-code L-7B6409FD`

## Phase 2 — Remediate

- Capacity errors: try different AZ or instance class
- Parameter group mismatch: create group with correct family (e.g., mysql8.0)
- Subnet group: ensure subnets in 2+ AZs for Multi-AZ
- Storage: verify IOPS/storage type valid for instance class

## Common Issues

- symptoms: "InsufficientDBInstanceCapacity"
  resolution: "Try a different AZ or instance class."

- symptoms: "Parameter group family mismatch"
  resolution: "Create parameter group with correct family (mysql5.7 vs mysql8.0)."

## Safety Ratings
- GREEN: describe-events, describe-db-engine-versions, describe-orderable-db-instance-options, describe-db-subnet-groups — read-only inspection
- YELLOW: modify-db-instance, modify-db-parameter-group — recoverable configuration changes
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Multi-AZ instance"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, KMS key IDs
- MEDIUM: engine versions, instance class availability, subnet group configuration

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix launch issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

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
