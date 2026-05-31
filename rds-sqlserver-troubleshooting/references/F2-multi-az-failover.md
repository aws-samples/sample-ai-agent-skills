---
title: "F2 — Multi-AZ Failover"
description: "Diagnose Multi-AZ failover events and issues for RDS SQL Server"
status: active
severity: CRITICAL
triggers:
  - "Multi-AZ"
  - "failover"
  - "Always On"
  - "AZ failure"
  - "unplanned failover"
  - "planned failover"
owner: devops-agent
objective: "Investigate Multi-AZ failover events and resolve post-failover issues"
context: "RDS SQL Server Multi-AZ uses Always On Availability Groups (SQL Server 2012+) managed by RDS. Failover is automatic on primary failure, AZ outage, instance class change, or manual reboot with failover. Typical failover time is 1-2 minutes. The DNS endpoint stays the same but the IP changes. Applications must handle reconnection. No Failover Cluster Instances (FCI)."
---

## Phase 1 — Triage

MUST:
- Check Multi-AZ status: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].{MultiAZ:MultiAZ,AZ:AvailabilityZone,SecondaryAZ:SecondaryAvailabilityZone}'`
- Check for failover events: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --event-categories failover --duration 4320`
- Check all recent events: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`
- Check instance status after failover: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].DBInstanceStatus'`

SHOULD:
- Check if failover was planned (reboot with failover, maintenance) or unplanned (AZ outage, instance failure)
- Verify application reconnected successfully
- Check CloudWatch for performance metrics around failover time
- Check error log for failover details:
  ```
  aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/ERRORLOG
  ```

MAY:
- Check AWS Health Dashboard for AZ-level events
- Review application logs for connection errors during failover
- Check Enhanced Monitoring for OS-level metrics around failover

## Phase 2 — Remediate

MUST:
- For post-failover connectivity: flush client DNS cache, verify application reconnected
- For application not reconnecting: implement connection retry logic with `MultiSubnetFailover=True` in connection string
- For performance degradation after failover: new primary may need buffer pool warm-up

SHOULD:
- Test failover manually to validate application resilience:
  ```
  aws rds reboot-db-instance \
    --db-instance-identifier <instance-id> \
    --force-failover
  ```
- Use `MultiSubnetFailover=True` in connection strings for faster failover:
  ```
  Server=<endpoint>;Database=MyDB;MultiSubnetFailover=True;
  ```
- Set up EventBridge rules for failover notifications:
  ```json
  {
    "source": ["aws.rds"],
    "detail-type": ["RDS DB Instance Event"],
    "detail": {"EventCategories": ["failover"]}
  }
  ```

MAY:
- Review failover frequency — frequent failovers may indicate underlying issues
- Consider read replicas for read scaling to reduce primary load

## Common Issues

- symptoms: "Application down for several minutes after failover"
  diagnosis: "Application caching old DNS/IP. Not implementing reconnection logic."
  resolution: "Use MultiSubnetFailover=True. Implement connection retry with exponential backoff."

- symptoms: "Performance degraded after failover"
  diagnosis: "New primary has cold buffer pool. Data cache needs to warm up."
  resolution: "Wait for cache warm-up. Consider pre-warming critical queries."

- symptoms: "Frequent unplanned failovers"
  diagnosis: "Underlying hardware or storage issues on primary AZ."
  resolution: "Check RDS events for root cause. Open AWS Support case if persistent."

- symptoms: "Multi-AZ conversion taking very long"
  diagnosis: "Initial sync of secondary requires full data copy."
  resolution: "Wait for sync to complete. Duration depends on database size and I/O capacity."

## Safety Ratings
- GREEN: describe-db-instances, describe-events, CloudWatch metrics — read-only inspection
- YELLOW: modify-db-instance, EventBridge rule creation — recoverable changes
- RED: reboot-db-instance --force-failover — high-impact operation, causes brief downtime

## Escalation Conditions
- "Database serves production traffic"
- "Failover caused application downtime exceeding SLA"
- "Frequent unplanned failovers indicating underlying issues"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, failover event details
- MEDIUM: CloudWatch metrics, instance status, AZ information

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix failover issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"
- "Restore from snapshot if failover resulted in data inconsistency"

## Output Format

```yaml
root_cause: "multi_az_failover — <specific_cause>"
evidence:
  - type: failover_event
    content: "<failover event details>"
  - type: instance_status
    content: "<current status and AZ>"
  - type: timeline
    content: "<failover start and completion times>"
severity: CRITICAL
mitigation:
  immediate: "Verify application connectivity and performance"
  long_term: "Implement failover testing and application resilience with MultiSubnetFailover"
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
