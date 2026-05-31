---
title: "C1 — OpenSearch Disk Watermarks"
description: "Diagnose and resolve disk watermark breaches causing shard allocation issues or write blocks"
status: active
severity: HIGH
triggers:
  - "disk watermark"
  - "low watermark"
  - "high watermark"
  - "flood stage"
  - "read_only_allow_delete"
  - "FreeStorageSpace"
owner: devops-agent
objective: "Identify disk watermark breaches and restore normal shard allocation and write operations"
context: "OpenSearch enforces three disk watermarks: low (85%) stops new shard allocation, high (90%) relocates shards away, flood stage (95%) sets indices to read-only. When flood stage is reached, indices get index.blocks.read_only_allow_delete set to true, blocking all writes. After freeing space, this block must be manually removed. Monitor FreeStorageSpace in CloudWatch."
---

## Phase 1 — Triage

MUST:
- Check disk allocation per node: `curl -XGET "https://<endpoint>/_cat/allocation?v"`
- Check free storage space: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name FreeStorageSpace --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`
- Check for read-only indices: `curl -XGET "https://<endpoint>/_all/_settings?pretty&filter_path=*.settings.index.blocks.read_only_allow_delete"`
- Check watermark settings: `curl -XGET "https://<endpoint>/_cluster/settings?include_defaults=true&flat_settings=true&filter_path=*.cluster.routing.allocation.disk.*"`
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`

SHOULD:
- Check largest indices: `curl -XGET "https://<endpoint>/_cat/indices?v&s=store.size:desc&h=index,store.size,docs.count" | head -20`
- Check shard distribution: `curl -XGET "https://<endpoint>/_cat/shards?v&h=index,shard,prirep,store,node&s=store:desc" | head -20`
- Check domain storage configuration: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.{EBSEnabled:EBSOptions.EBSEnabled,VolumeType:EBSOptions.VolumeType,VolumeSize:EBSOptions.VolumeSize}'`

MAY:
- Check for indices without ISM policies (no automatic cleanup)
- Check UltraWarm storage usage if enabled

## Phase 2 — Remediate

MUST:
- If flood stage reached: delete old/unnecessary indices to free space
- Remove read-only block after freeing space: `curl -XPUT "https://<endpoint>/_all/_settings" -H 'Content-Type: application/json' -d '{"index.blocks.read_only_allow_delete": null}'`
- If persistent: increase EBS volume size: `aws opensearch update-domain-config --domain-name <domain> --ebs-options EBSEnabled=true,VolumeType=gp3,VolumeSize=<new-size>`

SHOULD:
- Implement ISM policies to automatically delete or migrate old indices
- Set up CloudWatch alarms on FreeStorageSpace (alert at 75% usage)
- Migrate cold data to UltraWarm if enabled

MAY:
- Add data nodes to distribute storage load
- Force merge read-only indices to reclaim space from deleted documents
- Review and optimize index mappings to reduce storage footprint

## Common Issues

- symptoms: "Writes failing with 403 Forbidden and read_only_allow_delete"
  diagnosis: "Flood stage watermark (95%) reached. Indices set to read-only."
  resolution: "Delete old indices. Remove read-only block manually after freeing space."

- symptoms: "New indices not getting shards allocated"
  diagnosis: "Low watermark (85%) reached. No new shards allocated to full nodes."
  resolution: "Free disk space or add nodes. Check _cat/allocation for per-node usage."

- symptoms: "Shards relocating unexpectedly"
  diagnosis: "High watermark (90%) reached. OpenSearch relocating shards to less-full nodes."
  resolution: "Free disk space. Increase volume size. Add nodes for more total storage."

## Output Format

```yaml
root_cause: "disk_watermark — <specific_cause>"
evidence:
  - type: disk_allocation
    content: "<_cat/allocation output>"
  - type: free_storage
    content: "<FreeStorageSpace metric>"
  - type: read_only_blocks
    content: "<indices with read_only_allow_delete>"
severity: HIGH
mitigation:
  immediate: "Free disk space and remove read-only blocks"
  long_term: "Implement ISM policies, set up storage alarms, right-size volumes"
```


## Safety Ratings
```
safety_ratings:
  - "Check disk allocation and watermarks: GREEN — read-only diagnostics"
  - "Check for read-only blocks: GREEN — read-only inspection"
  - "Delete old indices: RED — permanently removes data"
  - "Remove read-only block: YELLOW — re-enables writes, ensure disk space was freed first"
  - "Increase EBS volume size: YELLOW — domain config change, 6-hour cooldown"
  - "Implement ISM policies: GREEN — adds lifecycle automation"
```

## Escalation Conditions
- Domain serves production search
- Flood stage reached — indices are read-only, writes blocked
- Fix requires blue/green deployment for volume or node changes
- ISM policies needed for long-term lifecycle management
- Multiple indices affected by read-only blocks

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Index names and sizes: data structure information"
    - "Disk allocation per node: infrastructure capacity"
    - "EBS volume configuration: storage details"
  handling: "Do not expose index names, sizes, or storage configuration externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER remove read-only blocks without first freeing sufficient disk space
- NEVER decrease EBS volume size (not supported)

## Phase 3 — Rollback
- If indices were deleted: restore from snapshot
- If read-only block was removed: block will be re-applied automatically if disk fills again
- If EBS volume was increased: CANNOT be decreased — volume increases are permanent
- If ISM policies were added: remove policies if they cause unintended deletions

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
