---
title: "Z1 — General ElastiCache Troubleshooting (Catch-All)"
description: "Fallback SOP for ElastiCache issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown ElastiCache issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step. Covers both Redis and Memcached engines."
---

## Phase 1 — Triage

MUST:
- Get cluster overview: `aws elasticache describe-cache-clusters --show-cache-node-info`
- Get replication group overview (Redis): `aws elasticache describe-replication-groups`
- Check cluster status: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].{Status:CacheClusterStatus,Engine:Engine,NodeType:CacheNodeType,EngineVersion:EngineVersion}'`
- Check key CloudWatch metrics:
  - `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name DatabaseMemoryUsagePercentage --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name Evictions --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CurrConnections --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Test basic connectivity: `redis-cli -h <endpoint> -p 6379 PING` (Redis) or `echo stats | nc <endpoint> 11211` (Memcached)
- Check recent events: `aws elasticache describe-events --source-type cache-cluster --duration 1440`

SHOULD:
- Check CloudTrail for recent configuration changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elasticache.amazonaws.com --max-results 20`
- Check parameter group: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].CacheParameterGroup'`
- Check security groups: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].SecurityGroups'`
- Check subnet group: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].CacheSubnetGroupName'`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Connection refused / timeout → Connectivity (A1-A3)
- High CPU / slow responses → Performance (B1-B4)
- Replication lag / failover → Replication (C1-C3)
- Scaling operations → Scaling (D1-D3)
- Evictions / OOM / fragmentation → Memory (E1-E3)
- AUTH / encryption / network → Security (F1-F3)
- Backup / restore → Backup (G1-G2)
- Slot migration / CROSSSLOT → Cluster Mode (H1-H3)
- Memcached auto-discovery / hashing → Memcached (I1-I2)
- Global Datastore → Global Datastore (J1-J2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- For Redis, run comprehensive INFO: `redis-cli -h <endpoint> -p 6379 INFO ALL`
- Check slow log: `redis-cli -h <endpoint> -p 6379 SLOWLOG GET 25`
- Check client list: `redis-cli -h <endpoint> -p 6379 CLIENT LIST`
- Review all CloudWatch metrics for anomalies

SHOULD:
- Check AWS Health Dashboard for ElastiCache service events
- Compare with a known-good cluster configuration
- Check pending service updates: `aws elasticache describe-service-updates`
- Check for pending parameter group changes

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps or specific SOP to follow

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Classify and pivot to specific SOP | GREEN | Diagnostic routing; no infrastructure change |
| Run comprehensive INFO ALL | GREEN | Read-only diagnostic; no data impact |
| Check slow log | GREEN | Read-only diagnostic; no data impact |
| Check client list | GREEN | Read-only diagnostic; no data impact |

## Escalation Conditions

- Issue cannot be classified into any specific failure domain
- Multiple concurrent issues detected across different domains
- Cluster in non-available status with no clear cause
- Issue persists after following specific SOP remediation
- Customer-facing impact with no clear root cause identified
- AWS Health Dashboard shows ElastiCache service events

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-cache-clusters` | MEDIUM | Exposes full cluster configuration |
| `get-metric-statistics` (all metrics) | LOW | Operational metrics only |
| `INFO ALL` | MEDIUM | Exposes server configuration, memory stats, and client info |
| `SLOWLOG GET` | MEDIUM | Exposes command patterns and key names |
| `CLIENT LIST` | MEDIUM | Exposes client connection details and addresses |
| `lookup-events` (CloudTrail) | MEDIUM | Exposes API calls and IAM principals |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) as a troubleshooting step
- NEVER suggest disabling AUTH during investigation
- NEVER suggest disabling encryption in transit during investigation
- NEVER suggest reducing node count during peak traffic
- NEVER make infrastructure changes during general troubleshooting without identifying root cause first

## Phase 3 — Rollback

General rollback guidance for unclassified issues:
1. If any configuration changes were made during investigation, document and revert them
2. If parameter group settings were modified, revert and reboot if required
3. If security group rules were changed for debugging, revert to previous rules
4. If the issue was escalated, provide all collected evidence to the escalation team
5. Verify all metrics return to baseline after any investigative changes are reverted

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<connectivity|performance|replication|scaling|memory|security|backup|cluster_mode|memcached|global_datastore|unknown>"
investigation_path: "cluster config → CloudWatch → events → <domain_classification>"
evidence:
  - type: cluster_config
    content: "<cluster configuration summary>"
  - type: cloudwatch
    content: "<key metrics>"
  - type: events
    content: "<relevant events>"
  - type: connectivity
    content: "<PING/connection test result>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest disabling AUTH"
  - "NEVER suggest public subnet placement"
