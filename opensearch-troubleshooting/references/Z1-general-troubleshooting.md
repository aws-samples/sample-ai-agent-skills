---
title: "Z1 — General OpenSearch Troubleshooting (Catch-All)"
description: "Fallback SOP for OpenSearch issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown OpenSearch issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step. Covers both managed OpenSearch domains and Serverless collections."
---

## Phase 1 — Triage

MUST:
- Check if managed domain or Serverless: `aws opensearch describe-domain --domain-name <domain>` or `aws opensearchserverless batch-get-collection --names <collection>`
- For managed domains:
  - Get domain overview: `aws opensearch describe-domain --domain-name <domain>`
  - Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`
  - Check node status: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,disk.used_percent,node.role"`
  - Check key CloudWatch metrics:
    - `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name ClusterStatus.red --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
    - `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
    - `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name FreeStorageSpace --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`
    - `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name CPUUtilization --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
  - Check recent events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=es.amazonaws.com --max-results 20`
- For Serverless:
  - Check collection status: `aws opensearchserverless batch-get-collection --names <collection>`
  - Check policies: `aws opensearchserverless list-security-policies --type encryption` and `--type network`
  - Check data access policies: `aws opensearchserverless list-access-policies --type data`

SHOULD:
- Check index listing: `curl -XGET "https://<endpoint>/_cat/indices?v&s=health,index"`
- Check shard allocation: `curl -XGET "https://<endpoint>/_cat/shards?v&h=index,shard,prirep,state,node&s=state"`
- Check domain configuration: `aws opensearch describe-domain-config --domain-name <domain>`
- Check access policy: `aws opensearch describe-domain-config --domain-name <domain> --query 'DomainConfig.AccessPolicies'`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Cluster health RED → Cluster Health (A1)
- Cluster health YELLOW → Cluster Health (A2)
- Master node issues → Cluster Health (A3-A4)
- Search latency / slow queries → Performance (B1)
- Indexing throughput issues → Performance (B2)
- JVM memory pressure → Performance (B3)
- GC pauses → Performance (B4)
- Disk watermarks / storage → Storage (C1-C2)
- UltraWarm/cold issues → Storage (C3)
- Unassigned shards → Shards (D1)
- Shard imbalance → Shards (D2)
- Too many shards → Shards (D3)
- Indexing failures → Indexing (E1-E3)
- Access denied / 403 → Access & Security (F1-F3)
- Snapshot/restore → Snapshots (G1-G3)
- Dashboards issues → Dashboards (H1-H2)
- Serverless issues → Serverless (I1-I2)
- ISM / rollover → ISM & Lifecycle (J1-J2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all node stats: `curl -XGET "https://<endpoint>/_nodes/stats?pretty"`
- Check hot threads: `curl -XGET "https://<endpoint>/_nodes/hot_threads"`
- Check cluster settings: `curl -XGET "https://<endpoint>/_cluster/settings?include_defaults=true&pretty"`
- Check pending tasks: `curl -XGET "https://<endpoint>/_cat/pending_tasks?v"`

SHOULD:
- Check AWS Health Dashboard for OpenSearch service events
- Compare with known-good domain configuration
- Check for pending domain updates: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.{Processing:Processing,UpgradeProcessing:UpgradeProcessing}'`
- Check service software update status: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.ServiceSoftwareOptions'`

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps or specific SOP to follow

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<cluster_health|performance|storage|shards|indexing|access_security|snapshots|dashboards|serverless|ism_lifecycle|unknown>"
investigation_path: "domain config → cluster health → CloudWatch → <domain_classification>"
evidence:
  - type: domain_config
    content: "<domain configuration summary>"
  - type: cluster_health
    content: "<cluster health status>"
  - type: cloudwatch
    content: "<key metrics>"
  - type: events
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings
```
safety_ratings:
  - "Check domain/collection status: GREEN — read-only API calls"
  - "Check cluster health and metrics: GREEN — read-only diagnostics"
  - "Check CloudTrail events: GREEN — read-only audit log query"
  - "Route to specific runbook: GREEN — diagnostic classification, no state change"
```

## Escalation Conditions
- Domain serves production search
- Issue cannot be classified into a specific failure domain
- Multiple failure domains affected simultaneously
- AWS service health issue suspected
- Fix requires blue/green deployment

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Domain configuration: infrastructure details"
    - "Cluster health and metrics: operational data"
    - "CloudTrail events: API call history"
    - "Node stats: cluster capacity details"
  handling: "System diagnostics may expose sensitive operational data. Mask domain names and endpoints in shared reports."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER make configuration changes without first classifying the failure domain
- NEVER apply fixes from multiple runbooks simultaneously

## Phase 3 — Rollback
- For general investigation: no rollback needed — all triage steps are read-only
- If routed to a specific runbook: follow that runbook's Phase 3 rollback procedures
- If configuration changes were made: revert each change individually

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
