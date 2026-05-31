---
title: "D1 — Node Failures"
description: "Diagnose and resolve Redshift compute node failures"
status: active
severity: CRITICAL
triggers:
  - "node failure"
  - "node down"
  - "hardware failure"
  - "node replacement"
  - "cluster degraded"
owner: devops-agent
objective: "Identify failed nodes and restore cluster to full health"
context: "Redshift automatically detects and replaces failed compute nodes. During replacement, the cluster operates in degraded mode with reduced performance. RA3 nodes recover faster because data is in managed storage. DC2 nodes require data replication from mirrors."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].{status:ClusterStatus,nodes:NumberOfNodes,health:ClusterHealthStatus}'`
- Check node health: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT node, is_diskfull, query_cpu, query_temp_blocks_to_disk FROM stv_node_storage_capacity"`
- Check cluster events for node issues: `aws redshift describe-events --source-identifier <cluster-id> --source-type cluster --duration 1440`
- Check CloudWatch health metric: `aws cloudwatch get-metric-statistics --namespace AWS/Redshift --metric-name HealthStatus --dimensions Name=ClusterIdentifier,Value=<cluster-id> --start-time $(date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Minimum`

SHOULD:
- Check if automatic node replacement is in progress
- Verify cluster availability during node replacement
- Check for recurring node failures (hardware pattern)

MAY:
- Check AWS Personal Health Dashboard for scheduled maintenance
- Review cluster audit logs for related events
- Check if the issue correlates with a recent resize operation

## Phase 2 — Remediate

MUST:
- If node replacement in progress: wait for automatic recovery (typically 5-15 minutes for RA3, longer for DC2)
- If cluster stuck in degraded state: contact AWS Support
- If recurring failures: consider migrating to a different node type or AZ

SHOULD:
- Monitor cluster events during recovery
- Verify data integrity after node replacement
- Check query performance after recovery

MAY:
- Enable cross-AZ snapshot copy for disaster recovery
- Consider multi-AZ deployment for critical workloads
- Review node type for known hardware issues

## Common Issues

- symptoms: "Cluster status shows 'hardware-failure' in events"
  diagnosis: "A compute node experienced hardware failure. Automatic replacement initiated."
  resolution: "Wait for automatic replacement. Monitor via describe-events. Cluster operates in degraded mode during replacement."

- symptoms: "Cluster unavailable after node failure"
  diagnosis: "Single-node cluster has no redundancy. Node failure causes full outage."
  resolution: "Use multi-node clusters for HA. Restore from snapshot if node replacement fails."

## Output Format

```yaml
root_cause: "Node failure — <specific_cause>"
evidence:
  - type: cluster_event
    content: "<node failure event details>"
severity: CRITICAL
mitigation:
  immediate: "Wait for automatic node replacement or contact AWS Support"
  long_term: "Use multi-node clusters, enable snapshots, consider multi-AZ"
```


## Safety Ratings
```
safety_ratings:
  - "Check cluster status: GREEN — read-only API call"
  - "Check node health: GREEN — read-only system table query"
  - "Check cluster events: GREEN — read-only API call"
  - "Wait for automatic node replacement: GREEN — no manual intervention"
  - "Contact AWS Support: GREEN — escalation, no state change"
  - "Restore from snapshot: RED — creates new cluster, requires connection string updates"
```

## Escalation Conditions
- Cluster serves production analytics
- Single-node cluster with no redundancy experiencing failure
- Multiple nodes failing simultaneously
- Node replacement taking longer than expected (>30 minutes for RA3)
- Cluster stuck in degraded state requiring AWS Support

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Cluster health status: operational state"
    - "Node storage capacity: infrastructure details"
    - "Cluster events: operational history"
  handling: "Do not expose cluster health details or node failure patterns externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER attempt manual node replacement — AWS handles this automatically
- NEVER resize a cluster during active node replacement
- NEVER delete a degraded cluster without first taking a snapshot

## Phase 3 — Rollback
- Node replacement is automatic — no manual rollback needed
- If cluster was restored from snapshot: update application connection strings to new endpoint
- If cluster was restored with different configuration: resize to match original configuration if needed
- If cross-AZ snapshot copy was used for DR: verify data consistency after restore

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
