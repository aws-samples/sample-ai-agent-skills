---
title: "D2 — Clustering Issues"
description: "Diagnose Snowball Edge cluster configuration and operation issues"
status: active
severity: HIGH
triggers: ["cluster error", "cluster node failed", "cluster quorum", "cluster creation failed"]
owner: devops-agent
objective: "Resolve Snowball Edge clustering issues including quorum, node failures, and data distribution"
context: "Snowball Edge clusters require minimum 5 devices. Clusters are immutable after creation. Data is distributed across nodes with redundancy."
---

## Phase 1 — Triage

MUST:
- List cluster jobs: `aws snowball list-cluster-jobs --cluster-id <cluster-id>`
- Describe cluster: `aws snowball describe-cluster --cluster-id <cluster-id>`
- Check each node status: `snowballEdge describe-device --endpoint https://<node-ip>:8243 --manifest-file <manifest> --unlock-code <code>`

SHOULD:
- Verify all cluster nodes are powered on and network-accessible
- Check cluster quorum status (majority of nodes must be available)

MAY:
- Review network connectivity between cluster nodes
- Check storage utilization across cluster nodes

## Phase 2 — Remediate

MUST:
- Ensure majority of cluster nodes (3 of 5) are available for quorum
- Fix network connectivity between cluster nodes
- Verify all nodes use the same cluster manifest and unlock code

SHOULD:
- Restart failed services on individual nodes
- Monitor cluster health after remediation

MAY:
- Redistribute workloads if a node has reduced capacity

## Safety Ratings

```
safety_ratings:
  - "list-cluster-jobs: GREEN — read-only cluster job listing"
  - "describe-cluster: GREEN — read-only cluster details"
  - "describe-device (per node): GREEN — read-only node status"
  - "associate-cluster: YELLOW — modifies cluster membership"
```

## Escalation Conditions

- Cluster loses quorum (fewer than 3 of 5 nodes available)
- A cluster node has hardware failure and cannot be recovered
- Data distribution across nodes is unbalanced
- Cluster creation fails due to device compatibility issues

## Data Sensitivity

- `aws snowball describe-cluster`: MEDIUM — reveals cluster configuration
- `snowballEdge describe-device`: MEDIUM — reveals node configuration
- Cluster data distribution: HIGH — reveals data placement across nodes

## Prohibited Actions

- NEVER attempt to add or remove devices from an existing cluster
- NEVER power off cluster nodes without understanding quorum impact
- NEVER assume a failed node's data is lost — cluster has redundancy
- NEVER create clusters with fewer than 5 devices

## Phase 3 — Rollback

1. If a cluster node was misconfigured, restart it and rejoin the cluster
2. If cluster services were stopped, restart them on all nodes
3. Verify cluster quorum is maintained after any changes
4. Check data integrity across cluster nodes
5. Document cluster node status and any remediation actions

## Common Issues

- symptoms: "Cluster operations fail with 'QuorumNotMet' error"
  diagnosis: "Fewer than 3 of 5 cluster nodes are available."
  resolution: "Restore network connectivity to failed nodes. Power on any offline nodes. Verify all nodes can communicate."

## Output Format

```yaml
root_cause: "clustering — <specific_cause>"
evidence:
  - type: cluster_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore cluster quorum"
  long_term: "Monitor cluster health"
```

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
