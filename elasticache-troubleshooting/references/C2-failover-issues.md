---
title: "C2 — ElastiCache Failover Issues"
description: "Diagnose failover failures, slow failover, or unexpected failover behavior in Redis replication groups"
status: active
severity: CRITICAL
triggers:
  - "failover"
  - "Multi-AZ"
  - "primary failed"
  - "automatic failover"
  - "failover not working"
  - "promotion"
owner: devops-agent
objective: "Ensure failover works correctly and minimize downtime during primary node failures"
context: "ElastiCache Redis with Multi-AZ automatic failover promotes a read replica to primary when the primary fails. DNS propagation takes 30-60 seconds. Failover can be triggered automatically (node failure, AZ failure) or manually (test-failover). Failover requires at least one replica. Cluster mode enabled has per-shard failover. Applications must handle brief connection interruptions."
---

## Phase 1 — Triage

MUST:
- Check replication group status and Multi-AZ: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].{Status:Status,MultiAZ:MultiAZ,AutoFailover:AutomaticFailover}'`
- Check recent failover events: `aws elasticache describe-events --source-type replication-group --duration 1440`
- Check node status: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].NodeGroups[*].NodeGroupMembers[*].{Id:CacheClusterId,Role:CurrentRole,AZ:PreferredAvailabilityZone}'`
- Verify replicas exist: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].NodeGroups[*].NodeGroupMembers'`
- Check CloudTrail for failover API calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=TestFailover --max-results 10`

SHOULD:
- Check replication lag before failover: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name ReplicationLag --dimensions Name=CacheClusterId,Value=<replica-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Verify replicas are in different AZs for true Multi-AZ protection
- Check if the application handles connection errors with retry logic
- Verify DNS TTL settings in the application

MAY:
- Test failover manually: `aws elasticache test-failover --replication-group-id <repl-group-id> --node-group-id <node-group-id>`
- Check SNS notifications for failover events
- Review application logs for connection errors during failover window

## Phase 2 — Remediate

MUST:
- Enable Multi-AZ with automatic failover if not already enabled: `aws elasticache modify-replication-group --replication-group-id <repl-group-id> --multi-az-enabled --automatic-failover-enabled`
- Ensure at least one replica exists in a different AZ
- Implement connection retry logic with exponential backoff in the application

SHOULD:
- Set DNS cache TTL to 60 seconds or less in the application
- Configure SNS notifications for failover events
- Test failover periodically using test-failover API
- Minimize replication lag to reduce data loss during failover

MAY:
- Use cluster mode enabled for per-shard failover (limits blast radius)
- Implement circuit breaker pattern in the application
- Consider Global Datastore for cross-region failover

## Common Issues

- symptoms: "Failover not triggering when primary is unresponsive"
  diagnosis: "Multi-AZ automatic failover is not enabled on the replication group."
  resolution: "Enable Multi-AZ and automatic failover using modify-replication-group."

- symptoms: "Application errors for 60+ seconds during failover"
  diagnosis: "Application caches DNS and does not retry connections after failover."
  resolution: "Set DNS TTL to 60s. Implement retry logic. Use the primary endpoint (not node endpoint)."

- symptoms: "Data loss after failover"
  diagnosis: "Replication lag was high before failover. Writes to old primary were not replicated."
  resolution: "Monitor ReplicationLag. Minimize lag. Accept that async replication may lose recent writes."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Multi-AZ automatic failover | GREEN | Adds resilience; no immediate operational impact |
| Implement connection retry logic | GREEN | Application-level resilience improvement |
| Set DNS cache TTL to 60s | GREEN | Application/JVM configuration change |
| Configure SNS notifications | GREEN | Monitoring only; no operational impact |
| Test failover (test-failover API) | YELLOW | Causes actual failover; brief write unavailability |
| Manual failover in production | RED | Causes primary switch; brief downtime; potential data loss from replication lag |

## Escalation Conditions

- Failover not triggering during actual primary node failure
- Application errors exceeding 60 seconds during failover (DNS caching issue)
- Data loss detected after failover (high replication lag before failover)
- Failover required but no replicas available
- Repeated unexpected failovers (flapping)

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-replication-groups` (Multi-AZ, failover) | MEDIUM | Exposes cluster architecture and HA configuration |
| `describe-events` (failover events) | LOW | Operational events only |
| `get-metric-statistics` (ReplicationLag) | LOW | Operational metrics only |
| `lookup-events` (CloudTrail) | MEDIUM | Exposes API calls and IAM principals |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix failover issues
- NEVER suggest disabling AUTH during failover troubleshooting
- NEVER suggest disabling encryption in transit to simplify failover
- NEVER suggest reducing node count during peak traffic
- NEVER test failover on a production cluster without stakeholder approval and during a maintenance window

## Phase 3 — Rollback

If failover configuration changes cause issues:
1. If Multi-AZ was enabled and causes unexpected failovers, investigate root cause before disabling
2. If DNS TTL was changed, revert to previous value
3. If SNS notifications are generating noise, adjust or remove the notification configuration
4. If a test failover was executed, the new primary is now active — plan failback if needed by executing another failover
5. If connection retry logic causes issues, revert to previous connection handling

## Output Format

```yaml
root_cause: "failover — <specific_cause>"
evidence:
  - type: replication_group
    content: "<Multi-AZ and failover configuration>"
  - type: events
    content: "<failover events>"
  - type: node_status
    content: "<node roles and AZ placement>"
severity: CRITICAL
mitigation:
  immediate: "Enable Multi-AZ failover or fix application retry logic"
  long_term: "Implement failover testing, monitoring, and application resilience patterns"
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
