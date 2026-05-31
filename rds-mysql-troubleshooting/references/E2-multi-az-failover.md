---
title: "E2 — Multi-AZ Failover"
description: "Diagnose Multi-AZ failover events, duration, and post-failover issues"
status: active
severity: CRITICAL
triggers:
  - "failover"
  - "Multi-AZ"
  - "instance reboot"
  - "availability event"
---

## Phase 1 — Triage

MUST:
- Check for failover events: `aws rds describe-events --source-identifier <id> --source-type db-instance --duration 1440 --event-categories failover`
- Verify Multi-AZ status: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].MultiAZ'`
- Check instance status post-failover: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].DBInstanceStatus'`
- Review CloudWatch for the failover window: check `DatabaseConnections` drop and recovery

SHOULD:
- Check if failover was triggered by: AZ failure, instance failure, manual reboot with failover, or maintenance
- Verify DNS propagation: `nslookup <endpoint>` — should resolve to new AZ IP
- Check application reconnection behavior — connection pooling should handle failover
- Review Enhanced Monitoring for OS-level issues before failover

## Phase 2 — Remediate

- Ensure applications handle connection drops gracefully (retry logic)
- Set DNS TTL low on application side (RDS endpoint TTL is 5 seconds)
- For planned failover testing: `aws rds reboot-db-instance --db-instance-identifier <id> --force-failover`
- Post-failover performance: standby may have cold buffer pool — expect temporary performance dip
- If frequent unplanned failovers: check for storage issues, instance health, or AZ problems

## Safety Ratings
- GREEN: describe-events, describe-db-instances, CloudWatch DatabaseConnections metrics, nslookup — read-only inspection
- YELLOW: modify-db-instance, EventBridge rule creation — recoverable changes
- RED: reboot-db-instance --force-failover — high-impact operation, causes brief downtime

## Escalation Conditions
- "Database serves production traffic"
- "Failover caused application downtime exceeding SLA"
- "Frequent unplanned failovers indicating underlying issues"
- "Fix requires another failover"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, failover event details
- MEDIUM: CloudWatch metrics, instance status, AZ information

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix failover issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"
- "Restore from snapshot if failover resulted in data inconsistency"
- "Allow buffer pool warm-up time after failover before assessing performance"

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
