---
title: "D1 — Read Replica Issues"
description: "Diagnose read replica lag, failover, and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "replica lag"
  - "read replica"
  - "replication delay"
  - "stale reads"
  - "reader instance"
owner: devops-agent
objective: "Identify and resolve read replica issues on DocumentDB clusters"
context: "DocumentDB supports up to 15 read replicas per cluster. Replicas share the same storage volume as the writer but may have replication lag for in-memory cache updates. Replica lag is typically under 100ms but can increase under heavy write load."
---

## Phase 1 — Triage

MUST:
- Check replica lag: `aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name DBClusterReplicaLagMaximum --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Describe cluster instances: `aws docdb describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id>`
- Verify reader instances are in available state
- Check writer instance CPU and write throughput

SHOULD:
- Check per-instance replica lag: `aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name ReplicaLag --dimensions Name=DBInstanceIdentifier,Value=<reader-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Verify reader endpoint is being used for read operations
- Check reader instance CPU and memory utilization
- Review failover priority settings

MAY:
- Compare performance across reader instances
- Check for reader instance maintenance events
- Analyze read distribution across readers

## Phase 2 — Remediate

MUST:
- Address writer instance bottlenecks causing high replication lag
- Ensure reader instances are appropriately sized
- Verify application uses reader endpoint for read operations

SHOULD:
- Set failover priority for preferred failover targets
- Add reader instances to distribute read load
- Monitor replica lag with CloudWatch alarms

MAY:
- Implement read preference in the application driver
- Use custom endpoints for specific reader groups

## Common Issues

- symptoms: "Replica lag spikes during heavy write periods"
  diagnosis: "Writer instance is under heavy write load, causing cache invalidation delays on readers."
  resolution: "Scale up writer instance class. Optimize write patterns. Monitor lag during peak periods."

- symptoms: "Reader endpoint returns stale data"
  diagnosis: "Normal replica lag — reads from replicas may be slightly behind the writer."
  resolution: "Use writer endpoint for reads requiring strong consistency. Accept eventual consistency for reader endpoint."

- symptoms: "Reader instance not receiving traffic"
  diagnosis: "Application is not using the reader endpoint or read preference is set to primary."
  resolution: "Configure application to use the reader endpoint. Set read preference to secondaryPreferred."

## Output Format

```yaml
root_cause: "read_replica — <specific_cause>"
evidence:
  - type: replica_lag
    content: "<replica lag metrics>"
  - type: instances
    content: "<reader instance status and metrics>"
  - type: endpoints
    content: "<endpoint configuration and usage>"
severity: MEDIUM
mitigation:
  immediate: "Address writer bottleneck or reader configuration"
  long_term: "Implement replica lag monitoring and read distribution"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Replica configuration affects data consistency |
| Blast Radius | MEDIUM — Replica issues affect read traffic distribution |
| Rollback Complexity | LOW — Replica configuration changes are non-destructive |
| Compliance Scope | MEDIUM — Read consistency requirements may be compliance-relevant |

## Escalation Conditions

- Escalate to application teams if replica lag causes stale read issues
- Escalate to operations if replica instances are unhealthy
- Escalate to DBA team if failover priority needs adjustment

## Data Sensitivity

- **Replica lag data** reveals write load patterns
- **Endpoint configuration** affects data consistency guarantees

## PROHIBITED Actions

- Do NOT use reader endpoint for operations requiring strong consistency
- Do NOT remove all reader instances without understanding the impact on read traffic
- Do NOT change failover priority without understanding the promotion implications

## Phase 3 — Rollback

- Restore previous failover priority settings
- Re-add reader instances if they were removed
- Revert application read preference settings if they were changed

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
