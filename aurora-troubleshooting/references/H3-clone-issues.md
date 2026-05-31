---
title: "H3 — Clone Issues"
description: "Diagnose Aurora cluster cloning issues"
status: active
severity: MEDIUM
triggers:
  - "clone"
  - "copy-on-write"
  - "clone creation"
  - "clone performance"
  - "clone storage"
owner: devops-agent
objective: "Identify and resolve Aurora cluster cloning issues"
context: "Aurora cloning uses copy-on-write protocol. Clone creation is near-instant regardless of database size. The clone shares storage pages with the source until pages are modified. Clone is a full independent cluster. Storage costs increase as the clone diverges from the source. Clones are useful for testing, development, and data analysis."
---

## Phase 1 — Triage

MUST:
- Check clone cluster status:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <clone-cluster-id>
  ```
- Check clone creation events:
  ```
  aws rds describe-events --source-identifier <clone-cluster-id> --source-type db-cluster --duration 1440
  ```
- Verify source cluster status:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <source-cluster-id> --query 'DBClusters[0].Status'
  ```
- Check clone storage usage:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeBytesUsed \
    --dimensions Name=DBClusterIdentifier,Value=<clone-cluster-id> ...
  ```

SHOULD:
- Compare clone and source storage usage over time (divergence)
- Check if clone has appropriate instance class and count
- Verify clone parameter groups are correct (clones inherit source parameter groups)

MAY:
- Check cross-account clone permissions if cloning across accounts
- Review clone lifecycle (when to delete clones to save costs)

## Phase 2 — Remediate

MUST:
- For clone creation failure: verify source cluster is in available state
- For clone performance issues: ensure clone instances are appropriately sized
- Understand that clone storage grows as data diverges from source

SHOULD:
- Create clones during low-traffic periods (minimal impact on source but best practice)
- Modify clone parameter groups if different settings are needed for testing
- Delete clones when no longer needed to avoid storage cost growth

MAY:
- Automate clone creation for periodic testing environments
- Use clones for blue/green deployment testing

## Common Issues

- symptoms: "Clone creation failed"
  diagnosis: "Source cluster not in available state, or insufficient permissions."
  resolution: "Verify source cluster status. Check IAM permissions for RestoreDBClusterToPointInTime."

- symptoms: "Clone storage costs growing rapidly"
  diagnosis: "Heavy write activity on clone causes copy-on-write page allocation."
  resolution: "Expected behavior. Delete clone when no longer needed. Use snapshots for long-term storage."

- symptoms: "Clone has different performance than source"
  diagnosis: "Clone instance class is different, or buffer pool/shared_buffers is cold."
  resolution: "Use same instance class. Allow time for cache warming."

## Safety Ratings
- GREEN: describe-db-clusters, describe-events, CloudWatch VolumeBytesUsed metrics — read-only inspection
- YELLOW: restore-db-cluster-to-point-in-time (clone creation), modify-db-cluster — recoverable operations
- RED: delete-db-cluster (clone deletion) — destructive but only affects the clone, not the source

## Escalation Conditions
- "Source database serves production traffic"
- "Clone creation failing due to source cluster issues"
- "Clone storage costs growing unexpectedly"
- "Fix involves modifying source cluster configuration"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings (clones inherit source credentials)
- HIGH: clone contains full copy of production data (same sensitivity as source)
- MEDIUM: clone storage metrics, parameter group configuration

## Prohibited Actions
- "NEVER suggest deleting the source cluster to fix clone issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest using clones as production databases without proper configuration review"
- "NEVER suggest sharing clone credentials without reviewing access controls"

## Phase 3 — Rollback
- "Delete clone if it is no longer needed to stop storage cost growth"
- "If clone has incorrect parameter groups, modify them rather than recreating"
- "If clone performance is poor, scale up clone instances rather than recreating"

## Output Format

```yaml
root_cause: "clone_issue — <specific_cause>"
evidence:
  - type: cluster_status
    content: "<clone and source cluster status>"
  - type: storage
    content: "<VolumeBytesUsed for clone>"
  - type: events
    content: "<clone creation events>"
severity: MEDIUM
mitigation:
  immediate: "Resolve clone creation or performance issue"
  long_term: "Implement clone lifecycle management and cost monitoring"
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
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"
