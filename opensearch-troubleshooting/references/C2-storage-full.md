---
title: "C2 — OpenSearch Storage Full"
description: "Diagnose and resolve storage capacity exhaustion on OpenSearch domains"
status: active
severity: CRITICAL
triggers:
  - "storage full"
  - "no space"
  - "disk full"
  - "volume full"
  - "FreeStorageSpace low"
  - "cannot allocate"
owner: devops-agent
objective: "Free storage space and prevent future storage exhaustion"
context: "OpenSearch domains use EBS volumes for data storage. When storage is exhausted, the cluster enters flood stage watermark, indices become read-only, and indexing stops. Storage consumption grows from new documents, replicas, segment merges (temporary), and transaction logs. Deleted documents don't free space until segments are merged. EBS volume size can be increased but not decreased."
---

## Phase 1 — Triage

MUST:
- Check free storage: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name FreeStorageSpace --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`
- Check per-node allocation: `curl -XGET "https://<endpoint>/_cat/allocation?v"`
- Check largest indices: `curl -XGET "https://<endpoint>/_cat/indices?v&s=store.size:desc&h=index,store.size,pri.store.size,docs.count,docs.deleted" | head -20`
- Check domain volume config: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.{EBSEnabled:EBSOptions.EBSEnabled,VolumeType:EBSOptions.VolumeType,VolumeSize:EBSOptions.VolumeSize,InstanceCount:ClusterConfig.InstanceCount}'`
- Check for read-only blocks: `curl -XGET "https://<endpoint>/_all/_settings?pretty&filter_path=*.settings.index.blocks.read_only_allow_delete"`

SHOULD:
- Check deleted document count (space not reclaimed until merge): `curl -XGET "https://<endpoint>/_cat/indices?v&h=index,docs.count,docs.deleted,store.size&s=docs.deleted:desc" | head -20`
- Check replica count per index: `curl -XGET "https://<endpoint>/_cat/indices?v&h=index,pri,rep,store.size&s=store.size:desc" | head -20`
- Check ISM policies: `curl -XGET "https://<endpoint>/_plugins/_ism/policies?pretty"`

MAY:
- Check UltraWarm usage if enabled: `curl -XGET "https://<endpoint>/_cat/indices?v&h=index,store.size&s=store.size:desc&expand_wildcards=all"`
- Check snapshot repository for backup before deletion

## Phase 2 — Remediate

MUST:
- Delete old/unnecessary indices: `curl -XDELETE "https://<endpoint>/<old-index>"`
- Remove read-only blocks: `curl -XPUT "https://<endpoint>/_all/_settings" -H 'Content-Type: application/json' -d '{"index.blocks.read_only_allow_delete": null}'`
- Increase EBS volume size if deletion is insufficient: `aws opensearch update-domain-config --domain-name <domain> --ebs-options EBSEnabled=true,VolumeType=gp3,VolumeSize=<new-size>`

SHOULD:
- Force merge indices with many deleted documents: `curl -XPOST "https://<endpoint>/<index>/_forcemerge?max_num_segments=1"`
- Reduce replica count on non-critical indices: `curl -XPUT "https://<endpoint>/<index>/_settings" -H 'Content-Type: application/json' -d '{"index":{"number_of_replicas":0}}'`
- Implement ISM policies for automatic index cleanup

MAY:
- Migrate old indices to UltraWarm if enabled
- Add data nodes for more total storage capacity
- Compress indices using best_compression codec for new indices

## Common Issues

- symptoms: "Storage full despite deleting documents"
  diagnosis: "Deleted documents occupy space until segment merge. docs.deleted count is high."
  resolution: "Force merge the index to reclaim space from deleted documents."

- symptoms: "Storage growing faster than expected"
  diagnosis: "High replica count multiplies storage. Each replica is a full copy."
  resolution: "Reduce replicas on non-critical indices. Implement ISM for lifecycle management."

- symptoms: "Cannot increase volume size (recently modified)"
  diagnosis: "EBS volume modifications have a 6-hour cooldown period."
  resolution: "Wait for cooldown. Delete indices to free immediate space."

## Output Format

```yaml
root_cause: "storage_full — <specific_cause>"
evidence:
  - type: free_storage
    content: "<FreeStorageSpace and per-node allocation>"
  - type: largest_indices
    content: "<top indices by size>"
  - type: volume_config
    content: "<EBS volume type and size>"
severity: CRITICAL
mitigation:
  immediate: "Delete old indices and remove read-only blocks"
  long_term: "Implement ISM policies, right-size volumes, set up storage alarms"
```


## Safety Ratings
```
safety_ratings:
  - "Check storage metrics and allocation: GREEN — read-only diagnostics"
  - "Check largest indices: GREEN — read-only inspection"
  - "Delete old indices: RED — permanently removes data"
  - "Remove read-only blocks: YELLOW — re-enables writes"
  - "Increase EBS volume: YELLOW — domain config change, 6-hour cooldown"
  - "Force merge to reclaim deleted docs space: YELLOW — resource-intensive I/O"
  - "Reduce replica count: YELLOW — reduces data redundancy"
```

## Escalation Conditions
- Domain serves production search
- Storage exhausted — all writes blocked
- Fix requires blue/green deployment for volume or node changes
- EBS volume recently modified (6-hour cooldown)
- Large indices need deletion requiring business approval

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Index names and sizes: data structure"
    - "Storage configuration: infrastructure details"
    - "Deleted document counts: data lifecycle information"
  handling: "Do not expose index names or storage details externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER remove read-only blocks without freeing disk space first
- NEVER set replicas to 0 on all production indices simultaneously

## Phase 3 — Rollback
- If indices were deleted: restore from snapshot
- If EBS volume was increased: CANNOT be decreased
- If replicas were reduced: increase replicas back to original count
- If force merge was performed: non-destructive, no rollback needed

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
