---
title: "A3 — Storage Issues"
description: "Diagnose DocumentDB storage capacity and growth problems"
status: active
severity: MEDIUM
triggers:
  - "storage full"
  - "storage growing"
  - "disk space"
  - "storage capacity"
  - "volume usage"
owner: devops-agent
objective: "Identify and resolve DocumentDB storage capacity and growth issues"
context: "DocumentDB storage auto-scales up to 128 TiB (standard) or 4 PiB (elastic). Storage is billed on high-water mark and does not immediately shrink when data is deleted. Storage growth can be caused by data growth, index overhead, change stream logs, or orphaned data."
---

## Phase 1 — Triage

MUST:
- Check storage usage: `aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name VolumeBytesUsed --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 3600 --statistics Maximum`
- Describe the cluster for storage configuration: `aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>`
- Check storage growth trend over time
- Verify change stream log retention setting in parameter group

SHOULD:
- Connect to cluster and check collection sizes: `mongo --tls --host <endpoint> --eval "db.stats()"`
- Check index sizes: `mongo --tls --host <endpoint> --eval "db.collection.stats().indexSizes"`
- Review change_stream_log_retention_duration parameter
- Check for large or growing collections

MAY:
- Analyze data distribution across collections
- Check for orphaned or temporary data
- Review TTL index effectiveness

## Phase 2 — Remediate

MUST:
- Identify the largest consumers of storage (data, indexes, change stream logs)
- Reduce change stream retention if logs are consuming excessive storage
- Delete unnecessary data or collections

SHOULD:
- Implement TTL indexes for automatic data expiration
- Optimize indexes to reduce index storage overhead
- Monitor storage growth trends proactively

MAY:
- Migrate to a new cluster to reclaim high-water mark storage
- Implement data archival strategies

## Common Issues

- symptoms: "Storage keeps growing despite deleting data"
  diagnosis: "DocumentDB storage is billed on high-water mark. Deleted data does not immediately reduce storage."
  resolution: "Storage reclaims gradually over time. For immediate reclamation, migrate data to a new cluster."

- symptoms: "Unexpected storage growth from change stream logs"
  diagnosis: "Change stream log retention is set to maximum (7 days) consuming significant storage."
  resolution: "Reduce change_stream_log_retention_duration parameter to match actual consumer needs."

- symptoms: "Index storage larger than data storage"
  diagnosis: "Too many indexes or indexes on large fields consuming excessive storage."
  resolution: "Review and remove unused indexes. Use db.collection.getIndexes() to audit."

## Output Format

```yaml
root_cause: "storage_issue — <specific_cause>"
evidence:
  - type: storage_metrics
    content: "<VolumeBytesUsed trend>"
  - type: collection_stats
    content: "<collection and index sizes>"
  - type: parameters
    content: "<change stream retention and other settings>"
severity: MEDIUM
mitigation:
  immediate: "Delete unnecessary data or reduce change stream retention"
  long_term: "Implement TTL indexes and storage monitoring"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Storage analysis may reveal data volume and growth patterns |
| Blast Radius | MEDIUM — Storage exhaustion can cause cluster unavailability |
| Rollback Complexity | MEDIUM — Storage high-water mark doesn't decrease immediately |
| Compliance Scope | MEDIUM — Data retention and storage management affect compliance |

## Escalation Conditions

- Escalate to operations if storage is approaching limits
- Escalate to DBA team if change stream logs are consuming excessive storage
- Escalate to data owners before deleting data to free storage

## Data Sensitivity

- **Storage metrics** reveal data volume — operational data
- **Collection statistics** may reveal data distribution patterns

## PROHIBITED Actions

- Do NOT delete production data without data owner approval
- Do NOT reduce change stream retention below consumer requirements
- Do NOT ignore storage growth trends — they indicate capacity planning needs

## Phase 3 — Rollback

- Restore deleted data from snapshots if deletion was incorrect
- Increase change stream retention if it was reduced too aggressively
- Note: storage high-water mark does not decrease immediately after data deletion

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
