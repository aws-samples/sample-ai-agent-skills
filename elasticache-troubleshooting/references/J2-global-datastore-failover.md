---
title: "J2 — Global Datastore Failover Issues"
description: "Diagnose Global Datastore cross-region failover problems"
status: active
severity: CRITICAL
triggers:
  - "Global Datastore failover"
  - "cross-region failover"
  - "promote secondary"
  - "region failover"
  - "global failover"
owner: devops-agent
objective: "Successfully execute Global Datastore cross-region failover and resolve failover issues"
context: "Global Datastore failover promotes a secondary region to become the new primary. This is a MANUAL operation — not automatic. The old primary becomes a secondary (if still available). Failover requires calling failover-global-replication-group. During failover, writes are briefly unavailable. The secondary region must be healthy and have low replication lag for minimal data loss. Requires Redis 5.0.6+ and cluster mode enabled."
---

## Phase 1 — Triage

MUST:
- Check Global Datastore status: `aws elasticache describe-global-replication-groups --global-replication-group-id <global-id>`
- Check member regions and roles: `aws elasticache describe-global-replication-groups --global-replication-group-id <global-id> --query 'GlobalReplicationGroups[*].Members'`
- Check secondary region health: `aws elasticache describe-replication-groups --replication-group-id <secondary-repl-group-id> --region <secondary-region>`
- Check replication lag before failover: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name ReplicationLag --dimensions Name=CacheClusterId,Value=<secondary-cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum --region <secondary-region>`
- Check recent events: `aws elasticache describe-events --source-type global-replication-group --duration 1440`

SHOULD:
- Verify the secondary region has sufficient capacity
- Check if any modifications are in progress on either region
- Verify application DNS/endpoint configuration for the secondary region
- Check CloudTrail for failover API calls

MAY:
- Check AWS Health Dashboard for regional issues
- Verify the application can switch to the secondary region endpoint
- Review disaster recovery runbook and RTO/RPO requirements

## Phase 2 — Remediate

MUST:
- Execute failover: `aws elasticache failover-global-replication-group --global-replication-group-id <global-id> --primary-region <secondary-region> --primary-replication-group-id <secondary-repl-group-id>`
- Update application connection strings to use the new primary region endpoint
- Monitor the failover operation via describe-global-replication-groups

SHOULD:
- Minimize replication lag before failover to reduce data loss
- Test failover periodically in non-production environments
- Document the failover procedure and endpoint changes
- Set up DNS-based routing (Route 53) for automatic endpoint switching

MAY:
- Implement application-level failover logic
- Configure health checks to detect primary region failures
- Plan for failback after the original primary region recovers

## Common Issues

- symptoms: "Failover fails with InvalidGlobalReplicationGroupState"
  diagnosis: "Global Datastore is not in 'available' status. Another operation is in progress."
  resolution: "Wait for current operations to complete. Check status with describe-global-replication-groups."

- symptoms: "Data loss after failover"
  diagnosis: "Replication lag was high before failover. Recent writes to old primary were not replicated."
  resolution: "Minimize replication lag before planned failover. Accept that async replication may lose recent writes."

- symptoms: "Application cannot connect after failover"
  diagnosis: "Application is still using the old primary region endpoint."
  resolution: "Update connection strings to the new primary region. Use Route 53 for DNS-based failover."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Execute failover (failover-global-replication-group) | RED | Switches primary region; brief write unavailability; potential data loss |
| Update application connection strings | YELLOW | Application change; must coordinate with all consumers |
| Monitor failover via describe-global-replication-groups | GREEN | Monitoring only; no operational impact |
| Set up DNS-based routing (Route 53) | YELLOW | DNS infrastructure change; affects traffic routing |
| Test failover in non-production | GREEN | Verification step; no production impact |

## Escalation Conditions

- Primary region outage requiring immediate failover decision
- Replication lag too high for acceptable RPO before failover
- Failover fails with InvalidGlobalReplicationGroupState
- Application unable to connect after failover (endpoints not updated)
- Data loss detected after failover due to replication lag
- Failback required after original primary region recovers

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-global-replication-groups` | MEDIUM | Exposes multi-region architecture and member roles |
| `get-metric-statistics` (ReplicationLag) | LOW | Operational metrics only |
| `describe-replication-groups` (secondary) | MEDIUM | Exposes cluster configuration |
| `describe-events` | LOW | Operational events only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) before or after failover
- NEVER suggest disabling AUTH during failover
- NEVER suggest disabling encryption in transit during failover
- NEVER suggest reducing node count during peak traffic
- NEVER execute a cross-region failover without confirming replication lag and data loss implications

## Phase 3 — Rollback

If Global Datastore failover causes issues:
1. If the new primary has issues, execute another failover back to the original region (if it has recovered): `aws elasticache failover-global-replication-group --global-replication-group-id <global-id> --primary-region <original-region> --primary-replication-group-id <original-repl-group-id>`
2. If application connection strings were updated, revert to original region endpoints
3. If DNS-based routing was configured, revert Route 53 records
4. If the original primary is still down, the new primary must remain active — focus on stabilizing it
5. Monitor replication lag and cluster health in all regions after rollback

## Output Format

```yaml
root_cause: "global_failover — <specific_cause>"
evidence:
  - type: global_datastore
    content: "<Global Datastore status and member regions>"
  - type: replication_lag
    content: "<pre-failover replication lag>"
  - type: failover_status
    content: "<failover operation status>"
severity: CRITICAL
mitigation:
  immediate: "Execute failover and update application endpoints"
  long_term: "Implement DNS-based failover, test regularly, minimize replication lag"
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest disabling AUTH"
  - "NEVER suggest public subnet placement"
