---
title: "F1 — Serverless v2 Scaling Issues"
description: "Diagnose Aurora Serverless v2 scaling problems"
status: active
severity: HIGH
triggers:
  - "Serverless scaling"
  - "ACU"
  - "ServerlessDatabaseCapacity"
  - "scaling too slow"
  - "not scaling up"
  - "scaling down"
owner: devops-agent
objective: "Identify and resolve Aurora Serverless v2 scaling issues"
context: "Aurora Serverless v2 scales between min and max ACU in 0.5 ACU increments. Each ACU provides ~2 GiB of memory. Scaling is not instant — there is latency during scale-up. Min ACU set too low causes performance issues. Max ACU limits the ceiling. Scaling is based on CPU, memory, and connection utilization."
---

## Phase 1 — Triage

MUST:
- Check current ACU capacity:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ServerlessDatabaseCapacity \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> \
    --start-time <start> --end-time <end> --period 60 --statistics Average Maximum Minimum
  ```
- Check ACU utilization:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ACUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> ...
  ```
- Check min/max ACU configuration:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].ServerlessV2ScalingConfiguration'
  ```
- Check CPU and memory during scaling events:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory ...
  ```

SHOULD:
- Check connection count during scaling:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> ...
  ```
- Review scaling events in RDS events:
  ```
  aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440
  ```
- Check if ACU is hitting max ceiling frequently

MAY:
- Compare ACU capacity with actual workload requirements
- Review cost implications of min/max ACU settings

## Phase 2 — Remediate

MUST:
- If scaling too slow: increase min ACU to handle baseline traffic without scaling delay
- If hitting max ACU: increase max ACU or optimize workload to reduce resource consumption
- If ACU utilization consistently high: consider provisioned instances instead of Serverless

SHOULD:
- Set min ACU high enough to avoid cold start for expected baseline traffic
- Monitor ACUUtilization to right-size min/max ACU settings
- Implement connection pooling (RDS Proxy) to reduce connection-driven scaling

MAY:
- Use mixed-configuration clusters (provisioned writer + Serverless readers or vice versa)
- Set up CloudWatch alarms for ACUUtilization approaching max

## Common Issues

- symptoms: "Performance degradation during traffic spikes"
  diagnosis: "Min ACU too low. Scaling up takes time and cannot keep up with sudden spikes."
  resolution: "Increase min ACU to handle expected baseline. Consider provisioned for predictable workloads."

- symptoms: "ACU at max but performance still poor"
  diagnosis: "Max ACU insufficient for workload, or queries are inefficient."
  resolution: "Increase max ACU. Optimize queries. Consider provisioned instances."

- symptoms: "Costs higher than expected with Serverless v2"
  diagnosis: "Min ACU set too high or workload keeps ACU elevated."
  resolution: "Review ACU usage patterns. Lower min ACU if cold start is acceptable. Compare with provisioned pricing."

## Safety Ratings
- GREEN: describe-db-clusters, CloudWatch ServerlessDatabaseCapacity/ACUUtilization/CPUUtilization/FreeableMemory/DatabaseConnections metrics, describe-events — read-only inspection
- YELLOW: modify-db-cluster (min/max ACU), modify-db-instance — recoverable configuration changes
- RED: delete-db-cluster, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Scaling issues causing application performance degradation"
- "Fix requires changing min/max ACU affecting cost and performance"
- "ACU consistently hitting maximum ceiling"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, scaling configuration
- MEDIUM: CloudWatch ACU metrics, CPU/memory metrics, connection counts
- MEDIUM: cost analysis data, workload patterns

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix scaling issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest setting min ACU to 0.5 for production workloads without understanding cold start impact"

## Phase 3 — Rollback
- "Revert min/max ACU changes if they cause performance or cost issues"
- "If scaling configuration change causes problems, restore previous ACU settings"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "serverless_scaling — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<ServerlessDatabaseCapacity, ACUUtilization metrics>"
  - type: scaling_config
    content: "<min/max ACU settings>"
  - type: performance
    content: "<CPU, memory, connections during scaling>"
severity: HIGH
mitigation:
  immediate: "Adjust min/max ACU settings"
  long_term: "Monitor ACU patterns and right-size configuration"
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
