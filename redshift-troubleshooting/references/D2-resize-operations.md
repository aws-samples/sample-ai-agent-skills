---
title: "D2 — Resize Operations"
description: "Diagnose and resolve Redshift cluster resize issues"
status: active
severity: HIGH
triggers:
  - "resize"
  - "elastic resize"
  - "classic resize"
  - "resize failed"
  - "resize stuck"
  - "add nodes"
owner: devops-agent
objective: "Identify resize operation issues and complete the resize successfully"
context: "Elastic resize adds/removes nodes in minutes (same node type). Classic resize changes node type and takes hours (data migration). RA3 elastic resize is fastest due to managed storage. During classic resize, the cluster is read-only."
---

## Phase 1 — Triage

MUST:
- Check resize status: `aws redshift describe-resize --cluster-identifier <cluster-id>`
- Check cluster status: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].{status:ClusterStatus,type:NodeType,nodes:NumberOfNodes}'`
- Check cluster events: `aws redshift describe-events --source-identifier <cluster-id> --source-type cluster --duration 1440`
- Verify target configuration is valid: check node type and count constraints

SHOULD:
- Check if elastic resize is available for the node type change
- Verify snapshot was taken before resize (automatic for classic resize)
- Check for pending maintenance that may conflict with resize

MAY:
- Check CloudTrail for ResizeCluster API call details
- Review resize history for the cluster
- Check if concurrent operations are blocking resize

## Phase 2 — Remediate

MUST:
- If resize failed: check error in events, fix the issue, and retry
- If resize stuck: contact AWS Support if stuck for more than expected duration
- If wrong target: cancel resize (if possible) and start with correct parameters
- If elastic resize not available: use classic resize (plan for downtime)

SHOULD:
- Use elastic resize when possible (minutes vs hours)
- Schedule classic resize during maintenance windows
- Take a manual snapshot before resize as a safety net

MAY:
- Consider RA3 migration for faster future resizes
- Use Redshift Serverless to avoid resize operations entirely
- Plan capacity with concurrency scaling instead of permanent resize

## Common Issues

- symptoms: "Elastic resize fails with 'unsupported node type change'"
  diagnosis: "Elastic resize only supports same node type with different count. Node type changes require classic resize."
  resolution: "Use classic resize for node type changes. Plan for hours of read-only downtime."

- symptoms: "Classic resize taking much longer than expected"
  diagnosis: "Large data volume requires extensive data migration to new nodes."
  resolution: "Wait for completion. Monitor progress via describe-resize. Consider RA3 for faster future resizes."

## Output Format

```yaml
root_cause: "Resize issue — <specific_cause>"
evidence:
  - type: resize_status
    content: "<describe-resize output>"
severity: HIGH
mitigation:
  immediate: "Fix resize parameters and retry, or wait for completion"
  long_term: "Migrate to RA3 for faster resizes, use elastic resize when possible"
```


## Safety Ratings
```
safety_ratings:
  - "Check resize status: GREEN — read-only API call"
  - "Check cluster events: GREEN — read-only API call"
  - "Elastic resize (same node type): YELLOW — brief unavailability (minutes), cluster restarts"
  - "Classic resize (different node type): RED — hours of read-only mode, data migration"
  - "Cancel resize: YELLOW — may leave cluster in intermediate state"
  - "Take manual snapshot before resize: GREEN — creates backup without cluster impact"
```

## Escalation Conditions
- Cluster serves production analytics
- Fix requires resize or restore
- Classic resize needed during business hours
- Resize failed and cluster is in an intermediate state
- Elastic resize not available for the desired configuration change

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Cluster configuration: node type, count, and capacity"
    - "Resize status: operational state details"
  handling: "Do not expose cluster sizing or resize details externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER start a classic resize without taking a snapshot first
- NEVER resize during active node replacement or maintenance
- NEVER cancel a classic resize mid-migration without understanding the consequences

## Phase 3 — Rollback
- If elastic resize was performed: resize back to original node count (triggers another brief restart)
- If classic resize was performed: restore from the pre-resize snapshot to revert to original configuration
- If resize failed: check cluster status — may need to restore from snapshot
- If resize completed but performance is worse: investigate query plans and distribution, or resize back

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
