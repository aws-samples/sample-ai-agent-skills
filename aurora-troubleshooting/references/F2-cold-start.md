---
title: "F2 — Serverless v2 Cold Start"
description: "Diagnose Aurora Serverless v2 cold start latency"
status: active
severity: MEDIUM
triggers:
  - "cold start"
  - "slow first connection"
  - "Serverless startup"
  - "initial latency"
  - "warming up"
owner: devops-agent
objective: "Identify and mitigate Aurora Serverless v2 cold start issues"
context: "Aurora Serverless v2 cold start occurs when the instance scales up from very low ACU (near minimum) after a period of low activity. The buffer pool/shared_buffers must be warmed, connections established, and compute resources allocated. Cold start is more noticeable with very low min ACU settings."
---

## Phase 1 — Triage

MUST:
- Check ACU capacity over time (look for drops to minimum):
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ServerlessDatabaseCapacity \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> \
    --start-time <24h-ago> --end-time <now> --period 300 --statistics Average Minimum
  ```
- Check min ACU setting:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].ServerlessV2ScalingConfiguration'
  ```
- Check latency during cold start periods:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadLatency \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteLatency ...
  ```
- Check connection count pattern:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> ...
  ```

SHOULD:
- Correlate cold start timing with application latency spikes
- Check FreeableMemory during scale-up (buffer pool warming):
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory ...
  ```
- Review application timeout settings (may need to be increased for cold start)

MAY:
- Implement a keep-alive query to prevent scaling to minimum
- Review if Serverless v2 is appropriate for the workload pattern

## Phase 2 — Remediate

MUST:
- Increase min ACU to avoid scaling down to very low values during idle periods
- Each ACU provides ~2 GiB memory — set min ACU high enough for buffer pool/shared_buffers baseline

SHOULD:
- Implement connection keep-alive or periodic health check queries to maintain minimum activity
- Configure application connection timeouts to tolerate cold start latency
- Use RDS Proxy to maintain warm connections during low-activity periods

MAY:
- Consider provisioned instances for workloads with strict latency requirements
- Use mixed-configuration clusters (provisioned writer + Serverless readers)

## Common Issues

- symptoms: "First queries of the day are very slow"
  diagnosis: "Serverless scaled to min ACU overnight. Buffer pool/shared_buffers cold."
  resolution: "Increase min ACU. Implement keep-alive queries."

- symptoms: "Latency spikes after idle periods"
  diagnosis: "ACU scaled down during idle. Scale-up latency on first requests."
  resolution: "Increase min ACU to maintain warm state."

- symptoms: "Connection timeout on first connection attempt"
  diagnosis: "Application timeout shorter than cold start scale-up time."
  resolution: "Increase connection timeout. Increase min ACU. Use RDS Proxy."

## Safety Ratings
- GREEN: describe-db-clusters, CloudWatch ServerlessDatabaseCapacity/ReadLatency/WriteLatency/DatabaseConnections/FreeableMemory metrics — read-only inspection
- YELLOW: modify-db-cluster (min ACU), application timeout configuration — recoverable changes
- RED: delete-db-cluster, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Cold start causing application timeouts and user-facing errors"
- "Fix requires changing min ACU affecting cost"
- "Fix requires application configuration changes"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, application configuration
- MEDIUM: CloudWatch ACU metrics, latency metrics, connection patterns
- MEDIUM: scaling configuration, cost analysis

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix cold start issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest setting min ACU to 0.5 for latency-sensitive production workloads"

## Phase 3 — Rollback
- "Revert min ACU changes if they cause unexpected cost increases"
- "Revert application timeout changes if they mask other issues"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "cold_start — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<ServerlessDatabaseCapacity showing drop to minimum>"
  - type: latency
    content: "<latency metrics during cold start>"
  - type: scaling_config
    content: "<min ACU setting>"
severity: MEDIUM
mitigation:
  immediate: "Increase min ACU"
  long_term: "Right-size min ACU based on workload patterns and latency requirements"
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
