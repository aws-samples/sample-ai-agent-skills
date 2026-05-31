---
title: "C3 — Listener Issues"
description: "Diagnose Oracle listener issues on RDS Oracle"
status: active
severity: MEDIUM
triggers:
  - "listener"
  - "ORA-12541"
  - "ORA-12518"
  - "ORA-12520"
  - "no listener"
  - "listener refused"
owner: devops-agent
objective: "Resolve Oracle listener-related connection issues on RDS Oracle"
context: "The Oracle listener on RDS is managed by AWS. You cannot start/stop/configure the listener directly. Listener issues on RDS typically indicate instance state problems (rebooting, failing over, storage full) rather than listener misconfiguration. The listener log is accessible via the RDS logs API."
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].DBInstanceStatus'`
- Check RDS events: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`
- Check listener log: `aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name trace/listener.log --number-of-lines 200`
- Check if Multi-AZ failover occurred: look for "Multi-AZ failover" in events

SHOULD:
- Check if instance is in maintenance: `aws rds describe-pending-maintenance-actions --resource-identifier <instance-arn>`
- Check storage space (listener fails if storage full): `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace ...`
- Verify endpoint DNS resolves to current IP (changes after failover)
- Check connection count — listener may refuse if at max:
  ```sql
  SELECT resource_name, current_utilization, limit_value
  FROM v$resource_limit WHERE resource_name = 'sessions';
  ```

MAY:
- Check alert log for instance-level errors: `aws rds download-db-log-file-portion --log-file-name trace/alert_<SID>.log`
- Verify from a different client to rule out client-side issues

## Phase 2 — Remediate

MUST:
- For instance not available: wait for RDS to complete the operation (reboot, failover, maintenance)
- For storage full causing listener failure: increase storage immediately
- For connection limit: kill idle sessions or increase processes parameter

SHOULD:
- After Multi-AZ failover: flush client DNS cache, reconnect (endpoint stays same but IP changes)
- Implement connection retry logic in applications
- Monitor instance status with CloudWatch alarms

MAY:
- Reboot instance if listener is unresponsive and instance status is "available":
  ```
  aws rds reboot-db-instance --db-instance-identifier <instance-id>
  ```

## Common Issues

- symptoms: "ORA-12541: TNS:no listener"
  diagnosis: "Listener not running. Instance may be rebooting, failing over, or in error state."
  resolution: "Check instance status. Wait for RDS to restore the instance. Do NOT try to start listener manually."

- symptoms: "ORA-12518: TNS:listener could not hand off client connection"
  diagnosis: "Listener running but cannot create new server process. Process limit reached or memory exhausted."
  resolution: "Check processes/sessions limits. Check FreeableMemory. Kill idle sessions."

- symptoms: "ORA-12520: TNS:listener could not find available handler"
  diagnosis: "All dispatchers/shared servers busy, or dedicated server processes at limit."
  resolution: "Increase processes parameter. Implement connection pooling."

- symptoms: "Listener errors after Multi-AZ failover"
  diagnosis: "Client cached old IP. DNS TTL not expired."
  resolution: "Flush DNS cache. Use the RDS endpoint (not IP). Set low DNS TTL in client."

## Safety Ratings
- GREEN: describe-db-instances, describe-events, describe-pending-maintenance-actions, CloudWatch FreeStorageSpace metrics, listener.log — read-only inspection
- YELLOW: reboot-db-instance, EXEC rdsadmin.rdsadmin_util.kill — recoverable but causes brief downtime
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Listener unresponsive and instance status shows available"
- "Fix requires rebooting the instance"
- "Storage full causing listener failure"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, listener log contents (may contain connection details)
- HIGH: alert log contents (may contain sensitive error details)
- MEDIUM: instance status, RDS events, maintenance actions

## Prohibited Actions
- "NEVER suggest starting/stopping the Oracle listener manually — RDS manages it"
- "NEVER suggest deleting a database instance to fix listener issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"

## Phase 3 — Rollback
- "If reboot does not resolve listener issues, check instance status and events"
- "Restore from snapshot if instance is in an unrecoverable state"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "listener — <specific_cause>"
evidence:
  - type: instance_status
    content: "<instance status>"
  - type: rds_events
    content: "<recent events>"
  - type: listener_log
    content: "<relevant listener log entries>"
severity: MEDIUM
mitigation:
  immediate: "Wait for RDS recovery or reboot if stuck"
  long_term: "Implement connection retry logic and monitoring"
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
