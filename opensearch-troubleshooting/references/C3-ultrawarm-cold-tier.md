---
title: "C3 — OpenSearch UltraWarm and Cold Tier Issues"
description: "Diagnose and resolve issues with UltraWarm and cold storage tiers"
status: active
severity: MEDIUM
triggers:
  - "UltraWarm"
  - "warm storage"
  - "cold storage"
  - "warm migration"
  - "cold migration"
  - "WarmStorageFreeSpace"
owner: devops-agent
objective: "Resolve UltraWarm and cold tier migration failures and access issues"
context: "UltraWarm provides cost-effective read-only storage for infrequently accessed data. Cold storage is even cheaper but requires migration back to warm before searching. Data flows: hot → warm → cold. UltraWarm indices are read-only — no indexing, updates, or deletes. Migration from hot to warm can take minutes to hours depending on index size. ISM policies automate tier transitions."
---

## Phase 1 — Triage

MUST:
- Check UltraWarm configuration: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.ClusterConfig.{WarmEnabled:WarmEnabled,WarmCount:WarmCount,WarmType:WarmType,ColdEnabled:ColdStorageOptions.Enabled}'`
- Check warm storage space: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name WarmFreeStorageSpace --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`
- Check migration status: `curl -XGET "https://<endpoint>/_ultrawarm/migration/_status?pretty"`
- Check warm indices: `curl -XGET "https://<endpoint>/_cat/indices?v&h=index,health,status,store.size&expand_wildcards=all" | grep warm`
- Check ISM policy status: `curl -XGET "https://<endpoint>/_plugins/_ism/explain/<index>?pretty"`

SHOULD:
- Check for failed migrations: `curl -XGET "https://<endpoint>/_ultrawarm/migration/_status?v"`
- Check warm node count and type: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,node.role"`
- Check cold storage indices: `curl -XGET "https://<endpoint>/_cold/indices/_search?pretty"`

MAY:
- Check ISM policy definition: `curl -XGET "https://<endpoint>/_plugins/_ism/policies/<policy-name>?pretty"`
- Review CloudTrail for migration API calls

## Phase 2 — Remediate

MUST:
- If migration failed: check error in migration status and retry: `curl -XPOST "https://<endpoint>/_ultrawarm/migration/<index>/_warm" -H 'Content-Type: application/json'`
- If warm storage full: migrate indices to cold or delete old warm indices
- If trying to write to warm index: writes are not supported — index to hot tier only

SHOULD:
- Use ISM policies to automate hot → warm → cold → delete lifecycle
- Monitor WarmFreeStorageSpace and set CloudWatch alarms
- Plan warm node capacity based on data retention requirements

MAY:
- Move cold indices back to warm for searching: `curl -XPOST "https://<endpoint>/_cold/indices/<index>/_warm"`
- Adjust ISM policy transition conditions (age, size, doc count)

## Common Issues

- symptoms: "Migration from hot to warm stuck or failed"
  diagnosis: "Insufficient warm storage space or warm node resources."
  resolution: "Free warm storage by deleting old warm indices or migrating to cold. Retry migration."

- symptoms: "Cannot search cold storage index"
  diagnosis: "Cold indices must be migrated back to warm before searching."
  resolution: "Move index from cold to warm first, then search."

- symptoms: "Write operations failing on warm index"
  diagnosis: "UltraWarm indices are read-only. Writes are not supported."
  resolution: "Index new data to hot tier. Use ISM to manage lifecycle transitions."

## Output Format

```yaml
root_cause: "ultrawarm_cold_tier — <specific_cause>"
evidence:
  - type: warm_config
    content: "<UltraWarm configuration and node count>"
  - type: migration_status
    content: "<migration status and errors>"
  - type: warm_storage
    content: "<WarmFreeStorageSpace metric>"
severity: MEDIUM
mitigation:
  immediate: "Resolve migration failure or free warm storage"
  long_term: "Implement ISM lifecycle policies, right-size warm tier capacity"
```


## Safety Ratings
```
safety_ratings:
  - "Check UltraWarm configuration and migration status: GREEN — read-only diagnostics"
  - "Check warm indices: GREEN — read-only inspection"
  - "Migrate index to warm: YELLOW — makes index read-only, migration takes time"
  - "Migrate index from cold to warm: YELLOW — consumes warm storage"
  - "Delete warm indices: RED — permanently removes data"
```

## Escalation Conditions
- Domain serves production search
- Migration failures blocking index lifecycle
- Warm storage full requiring cleanup or capacity increase
- Fix requires blue/green deployment to add warm nodes

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Index data in warm/cold tiers: historical business data"
    - "ISM policy configuration: lifecycle rules"
    - "Warm storage metrics: capacity information"
  handling: "Warm and cold tier data may contain historical sensitive data. Apply same access controls."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER attempt to write to UltraWarm indices (they are read-only)
- NEVER delete warm indices without confirming they are backed up

## Phase 3 — Rollback
- If index was migrated to warm: cannot easily move back to hot — reindex from source if needed
- If warm indices were deleted: restore from snapshot
- If ISM policy was modified: revert to previous policy definition
- If cold-to-warm migration was started: wait for completion or cancel if possible

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
