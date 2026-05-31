---
title: "D1 — Parameter Group Issues"
description: "Diagnose RDS MariaDB parameter group misconfiguration"
status: active
severity: MEDIUM
triggers:
  - "parameter group"
  - "pending-reboot"
---

## Phase 1 — Triage

MUST:
- Check parameter group: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].DBParameterGroups'`
- Check pending-reboot status
- List modified parameters: `aws rds describe-db-parameters --db-parameter-group-name <group> --source user`

SHOULD:
- Verify family matches engine version (mariadb10.6, mariadb10.11)
- Key MariaDB parameters: `innodb_buffer_pool_size`, `max_connections`, `thread_pool_size`, `slow_query_log`, `binlog_format`, `optimizer_switch`

## Phase 2 — Remediate

- Static parameters require reboot
- Dynamic parameters apply immediately
- Wrong family: create new group, associate, reboot


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
