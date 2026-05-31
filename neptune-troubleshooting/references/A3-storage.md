---
title: "A3 — Storage Issues"
description: "Diagnose Neptune storage capacity and performance problems"
status: active
severity: HIGH
triggers:
  - "storage full"
  - "storage growing"
  - "volume bytes"
  - "storage capacity"
owner: devops-agent
objective: "Identify and resolve Neptune storage issues"
context: "Neptune uses auto-scaling distributed storage (up to 128 TiB). Storage grows automatically but does not shrink when data is deleted. Storage performance is tied to volume IOPS."
---

## Phase 1 — Triage

MUST:
- Check storage usage: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name VolumeBytesUsed --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 3600 --statistics Average`
- Check volume IOPS: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name VolumeReadIOPs --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check write IOPS: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name VolumeWriteIOPs --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Describe cluster for storage configuration: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id>`

SHOULD:
- Monitor storage growth trend over time
- Check buffer cache hit ratio: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name BufferCacheHitRatio --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify no runaway data loading processes

MAY:
- Estimate data size with Gremlin: `curl -X POST https://<endpoint>:8182/gremlin -d '{"gremlin":"g.V().count()"}'`
- Check SPARQL triple count: `curl -X POST https://<endpoint>:8182/sparql -d 'query=SELECT (COUNT(*) AS ?count) WHERE { ?s ?p ?o }'`

## Phase 2 — Remediate

MUST:
- If storage is growing unexpectedly, identify the source of data growth
- If storage needs reclaiming, clone the cluster or restore from snapshot

SHOULD:
- Set up CloudWatch alarms for VolumeBytesUsed
- Review data retention policies

MAY:
- Consider partitioning data across multiple clusters
- Implement data archival strategies

## Common Issues

- symptoms: "Storage keeps growing after deleting data"
  diagnosis: "Neptune storage does not automatically shrink after data deletion."
  resolution: "Clone the cluster to reclaim space. Only active data is copied to the clone."

- symptoms: "High volume IOPS with slow queries"
  diagnosis: "Working set exceeds instance memory, causing excessive disk reads."
  resolution: "Scale to larger instance class with more memory for better buffer cache hit ratio."

- symptoms: "Approaching 128 TiB storage limit"
  diagnosis: "Data volume is approaching the maximum Neptune storage limit."
  resolution: "Archive old data, partition across clusters, or review data model efficiency."

## Output Format

```yaml
root_cause: "storage — <specific_cause>"
evidence:
  - type: storage_metrics
    content: "<VolumeBytesUsed, IOPS metrics>"
  - type: cluster
    content: "<cluster configuration>"
severity: HIGH
mitigation:
  immediate: "Address storage growth or reclaim space"
  long_term: "Implement storage monitoring and data lifecycle management"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — storage issues may involve data integrity concerns
- Service Disruption Risk: HIGH — storage failures can cause data loss
- Compliance Impact: HIGH — data integrity directly affects compliance

## Escalation Conditions
- Storage volume approaching maximum capacity
- Storage I/O errors detected in cluster logs
- Unexpected storage growth indicating data loading issues
- Storage encryption key becomes inaccessible
- Storage-related performance degradation

## Data Sensitivity
- Storage volume metrics — INTERNAL
- Data size and growth patterns — INTERNAL
- Encryption key configurations — CONFIDENTIAL
- Backup and snapshot data — CONFIDENTIAL
- Never expose storage volume identifiers externally

## Prohibited Actions
- Do NOT ignore storage capacity warnings
- Do NOT disable storage encryption on production clusters
- Do NOT delete snapshots that serve as the only backup
- Do NOT run bulk data operations without monitoring storage impact
- Do NOT rotate KMS keys without verifying Neptune can access the new key
- Do NOT bypass storage monitoring alerts

## Phase 3 — Rollback
- Restore from snapshot if data corruption is detected
- Revert bulk data loading operations that caused storage issues
- Re-enable storage monitoring if disabled during troubleshooting
- Verify data integrity after storage-related incidents
- Document storage growth patterns for capacity planning
- Update storage alerts and thresholds based on incident findings

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
