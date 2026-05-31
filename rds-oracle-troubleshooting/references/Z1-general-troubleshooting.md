---
title: "Z1 — General RDS Oracle Troubleshooting (Catch-All)"
description: "Fallback SOP for RDS Oracle issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown RDS Oracle issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws rds describe-db-instances --db-instance-identifier <instance-id>`
- Check recent events: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`
- Check key CloudWatch metrics:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections ...
  ```
- Check alert log: `aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name trace/alert_<SID>.log --number-of-lines 500`
- Check parameter group status: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].DBParameterGroups'`

SHOULD:
- Check Oracle-level health:
  ```sql
  SELECT status FROM v$instance;
  SELECT tablespace_name, ROUND(used_percent,2) AS pct_used FROM dba_tablespace_usage_metrics;
  SELECT resource_name, current_utilization, limit_value FROM v$resource_limit WHERE resource_name IN ('sessions','processes');
  SELECT event, wait_class, COUNT(*) FROM v$session WHERE status = 'ACTIVE' AND wait_class != 'Idle' GROUP BY event, wait_class ORDER BY COUNT(*) DESC;
  ```
- Check pending modifications: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].PendingModifiedValues'`
- Check pending maintenance: `aws rds describe-pending-maintenance-actions --resource-identifier <instance-arn>`
- Check option group: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].OptionGroupMemberships'`

MAY:
- Check Performance Insights for DB load trends
- Check Enhanced Monitoring for OS-level metrics
- Review CloudTrail for recent configuration changes

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Instance launch/config issues → Instance (A1-A3)
- CPU, memory, I/O, wait events → Performance (B1-B4)
- Connection failures, TNS, listener → Connectivity (C1-C3)
- Parameter group, init params, limits → Parameters (D1-D3)
- Backup, snapshot, PITR → Backup & Recovery (E1-E3)
- Read replica, Multi-AZ → Replication (F1-F2)
- Data Pump, DMS, character set → Migration (G1-G3)
- TDE, SSL, audit → Security (H1-H3)
- Upgrades, patching → Maintenance (I1-I2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all instance configurations systematically
- Review CloudTrail for recent API calls affecting the instance
- Check for ORA- errors in alert log and trace files
- Verify network connectivity and security group rules

SHOULD:
- Check AWS Health Dashboard for RDS service events
- Compare with a known-good instance configuration
- Check for recent changes (parameter group, option group, security group)

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings
- GREEN: describe-db-instances, describe-events, CloudWatch metrics, alert log, SELECT from v$instance/dba_tablespace_usage_metrics/v$resource_limit/v$session — read-only inspection
- YELLOW: modify-db-instance, modify-db-parameter-group — recoverable configuration changes
- RED: delete-db-instance, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Root cause cannot be identified — escalate to AWS Support"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Multi-AZ instance"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, alert log contents (may contain SQL text and sensitive data)
- HIGH: v$session details (contain active SQL text and connection details)
- MEDIUM: CloudWatch metrics, instance configuration, pending maintenance actions

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "Restore from snapshot if configuration change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<instance|performance|connectivity|parameters|backup|replication|migration|security|maintenance|unknown>"
investigation_path: "instance status → events → metrics → Oracle health → <domain_classification>"
evidence:
  - type: instance_config
    content: "<instance configuration summary>"
  - type: cloudwatch
    content: "<key metrics>"
  - type: oracle_health
    content: "<Oracle-level diagnostics>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
