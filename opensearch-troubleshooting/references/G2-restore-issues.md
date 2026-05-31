---
title: "G2 — OpenSearch Restore Issues"
description: "Diagnose and resolve snapshot restore failures"
status: active
severity: HIGH
triggers:
  - "restore failed"
  - "restore snapshot"
  - "restore error"
  - "snapshot restore"
  - "index already exists"
owner: devops-agent
objective: "Successfully restore indices from snapshots"
context: "Restoring from snapshots creates indices from a previously taken snapshot. The target domain must have sufficient storage and shard capacity. Indices with the same name must be closed or deleted before restore. Automated snapshots can only be restored to the same domain. Manual snapshots (S3) can be restored to any domain with access to the repository. Version compatibility must be maintained."
---

## Phase 1 — Triage

MUST:
- Check snapshot details: `curl -XGET "https://<endpoint>/_snapshot/<repo>/<snapshot>?pretty"`
- Check snapshot state: `curl -XGET "https://<endpoint>/_snapshot/<repo>/<snapshot>/_status?pretty"`
- Check if target indices exist: `curl -XGET "https://<endpoint>/_cat/indices/<index-pattern>?v"`
- Check available storage: `curl -XGET "https://<endpoint>/_cat/allocation?v"`
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`

SHOULD:
- Check snapshot indices list: `curl -XGET "https://<endpoint>/_snapshot/<repo>/<snapshot>?pretty" | grep -A 100 '"indices"'`
- Check domain version compatibility: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.EngineVersion'`
- Check shard count in snapshot vs available capacity

MAY:
- Check restore progress: `curl -XGET "https://<endpoint>/_cat/recovery?v&active_only=true"`
- Check for index template conflicts

## Phase 2 — Remediate

MUST:
- If index exists: close or delete before restore: `curl -XPOST "https://<endpoint>/<index>/_close"` or `curl -XDELETE "https://<endpoint>/<index>"`
- Restore specific indices: `curl -XPOST "https://<endpoint>/_snapshot/<repo>/<snapshot>/_restore" -H 'Content-Type: application/json' -d '{"indices":"index-1,index-2","ignore_unavailable":true}'`
- If insufficient storage: free space or increase volume size before restore

SHOULD:
- Restore to a different index name to avoid conflicts: `curl -XPOST "https://<endpoint>/_snapshot/<repo>/<snapshot>/_restore" -H 'Content-Type: application/json' -d '{"indices":"original-index","rename_pattern":"(.+)","rename_replacement":"restored-$1"}'`
- Monitor restore progress via _cat/recovery
- Verify data integrity after restore

MAY:
- Restore with modified settings (fewer replicas for faster restore): `curl -XPOST "https://<endpoint>/_snapshot/<repo>/<snapshot>/_restore" -H 'Content-Type: application/json' -d '{"indices":"my-index","index_settings":{"index.number_of_replicas":0}}'`
- Restore from a different domain's S3 repository by registering the same repo

## Common Issues

- symptoms: "index_already_exists_exception during restore"
  diagnosis: "Target index already exists. Must close or delete first."
  resolution: "Delete or close the existing index, or use rename_pattern to restore with a different name."

- symptoms: "Restore fails with insufficient storage"
  diagnosis: "Not enough free disk space on the target domain for the restored indices."
  resolution: "Free disk space, increase volume size, or restore fewer indices."

- symptoms: "Cannot restore automated snapshot to different domain"
  diagnosis: "Automated snapshots are stored internally and only accessible from the same domain."
  resolution: "Use manual snapshots (S3 repository) for cross-domain restore."

## Output Format

```yaml
root_cause: "restore_issue — <specific_cause>"
evidence:
  - type: snapshot_details
    content: "<snapshot state and indices>"
  - type: target_cluster
    content: "<available storage and cluster health>"
  - type: error
    content: "<restore error message>"
severity: HIGH
mitigation:
  immediate: "Fix restore prerequisites and retry"
  long_term: "Document restore procedures, test regularly, use manual snapshots for DR"
```


## Safety Ratings
```
safety_ratings:
  - "Check snapshot details and target cluster: GREEN — read-only diagnostics"
  - "Close existing index before restore: YELLOW — makes index temporarily unavailable"
  - "Delete existing index before restore: RED — permanently removes current data"
  - "Restore from snapshot: YELLOW — creates indices, consumes storage"
  - "Restore with rename: GREEN — creates new indices without affecting existing"
```

## Escalation Conditions
- Domain serves production search
- Restore needed for disaster recovery
- Insufficient storage for restored indices
- Version compatibility issues between snapshot and target domain
- Cross-domain restore requiring S3 repository access

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Snapshot data: historical index data"
    - "Restored indices: business search data"
    - "Repository configuration: S3 bucket and IAM details"
  handling: "Restored data may contain sensitive business content. Apply same access controls as original indices."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER delete existing indices before restore without confirming the snapshot contains the data
- NEVER restore automated snapshots to a different domain (not supported)

## Phase 3 — Rollback
- If existing index was deleted for restore: CANNOT be recovered if restore fails — always verify snapshot first
- If restore created wrong indices: delete the restored indices
- If restore with rename was used: delete the renamed indices if not needed
- If index was closed for restore: reopen the index if restore is abandoned

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling fine-grained access control"
  - "NEVER suggest public access domains"
  - "NEVER suggest disabling encryption at rest"
