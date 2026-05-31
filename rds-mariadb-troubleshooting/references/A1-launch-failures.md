---
title: "A1 — RDS MariaDB Launch Failures"
description: "Diagnose failures when creating or launching an RDS MariaDB instance"
status: active
severity: HIGH
triggers:
  - "launch failed"
  - "create db instance failed"
  - "InsufficientDBInstanceCapacity"
---

## Phase 1 — Triage

MUST:
- Check RDS events: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`
- Verify engine version: `aws rds describe-db-engine-versions --engine mariadb --query 'DBEngineVersions[].EngineVersion'`
- Check instance class availability: `aws rds describe-orderable-db-instance-options --engine mariadb --engine-version <version> --query 'OrderableDBInstanceOptions[].DBInstanceClass'`
- Verify subnet group spans 2+ AZs
- Confirm parameter group family matches engine (e.g., mariadb10.6)

SHOULD:
- Verify security group allows port 3306
- Check KMS key if encrypted
- Check service quotas

## Phase 2 — Remediate

- Capacity errors: try different AZ or instance class
- Parameter group mismatch: create group with correct family
- Subnet group: ensure 2+ AZs for Multi-AZ


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Database connection strings may contain credentials
- Query logs may contain sensitive application data
- Parameter groups may reveal security configurations
- Classification: CONFIDENTIAL - potential exposure of database credentials and application data

## Prohibited Actions

- NEVER force-reboot production RDS instances without approval
- NEVER modify parameter groups attached to production instances without testing
- NEVER delete automated backups or disable backup retention
- NEVER promote read replicas without confirming application readiness

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
