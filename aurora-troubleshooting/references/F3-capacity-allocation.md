---
title: "F3 — Serverless v2 Capacity Allocation"
description: "Diagnose Aurora Serverless v2 capacity allocation and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "capacity allocation"
  - "ACU configuration"
  - "min ACU"
  - "max ACU"
  - "Serverless v2 configuration"
  - "capacity not sufficient"
owner: devops-agent
objective: "Identify and resolve Aurora Serverless v2 capacity allocation issues"
context: "Aurora Serverless v2 capacity is configured with min and max ACU. Min ACU can be as low as 0.5. Max ACU can be up to the equivalent of the largest provisioned instance. Each ACU provides ~2 GiB of memory. Incorrect capacity allocation leads to performance issues, cold starts, or unnecessary costs."
---

## Phase 1 — Triage

MUST:
- Check current capacity configuration:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].ServerlessV2ScalingConfiguration'
  ```
- Check actual ACU usage over time:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ServerlessDatabaseCapacity \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> \
    --start-time <7-days-ago> --end-time <now> --period 3600 --statistics Average Maximum Minimum
  ```
- Check ACU utilization:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ACUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> ...
  ```
- Check if instances in the cluster are Serverless or provisioned:
  ```
  aws rds describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id> \
    --query 'DBInstances[].{Id:DBInstanceIdentifier,Class:DBInstanceClass}'
  ```

SHOULD:
- Calculate memory requirements: connections × per-connection memory + buffer pool/shared_buffers
- Compare ACU usage patterns with workload patterns (peak hours, batch jobs)
- Check if max ACU is being hit frequently (indicates undersizing)

MAY:
- Compare Serverless v2 cost with equivalent provisioned instance cost
- Review if mixed-configuration (provisioned + Serverless) would be more cost-effective

## Phase 2 — Remediate

MUST:
- For performance issues: increase min ACU to handle baseline without scaling delay
- For cost optimization: analyze ACU usage patterns and adjust min/max accordingly
- For capacity ceiling: increase max ACU if workload regularly hits the maximum

SHOULD:
- Set min ACU based on: (required memory in GiB) / 2 = min ACU
- Set max ACU based on peak workload requirements with headroom
- Modify capacity: `aws rds modify-db-cluster --db-cluster-identifier <cluster-id> --serverless-v2-scaling-configuration MinCapacity=<min>,MaxCapacity=<max>`

MAY:
- Use different capacity settings for writer vs reader Serverless instances
- Implement scheduled scaling adjustments for predictable workload patterns

## Common Issues

- symptoms: "Min ACU too low causing cold start and performance issues"
  diagnosis: "Min ACU set to 0.5 but workload needs at least 4 ACU baseline."
  resolution: "Increase min ACU to match baseline workload requirements."

- symptoms: "Max ACU too low for peak workload"
  diagnosis: "ACU hitting max ceiling during peak hours."
  resolution: "Increase max ACU. Optimize peak workload queries."

- symptoms: "Serverless v2 more expensive than provisioned"
  diagnosis: "Workload is steady-state, not variable. Serverless overhead not justified."
  resolution: "Consider switching to provisioned instances for steady workloads."

## Safety Ratings
- GREEN: describe-db-clusters, describe-db-instances, CloudWatch ServerlessDatabaseCapacity/ACUUtilization metrics — read-only inspection
- YELLOW: modify-db-cluster (min/max ACU) — recoverable configuration changes
- RED: delete-db-cluster, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires changing min/max ACU affecting cost and performance"
- "ACU consistently hitting maximum ceiling"
- "Capacity allocation change may impact application performance"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, instance class details
- MEDIUM: CloudWatch ACU metrics, capacity configuration, cost analysis
- MEDIUM: workload patterns, memory requirements

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix capacity issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest reducing max ACU below current workload requirements"

## Phase 3 — Rollback
- "Revert min/max ACU changes if they cause performance or cost issues"
- "If capacity change causes problems, restore previous ACU settings immediately"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "capacity_allocation — <specific_cause>"
evidence:
  - type: scaling_config
    content: "<min/max ACU settings>"
  - type: cloudwatch
    content: "<ACU usage patterns over time>"
  - type: cost_analysis
    content: "<Serverless vs provisioned cost comparison>"
severity: MEDIUM
mitigation:
  immediate: "Adjust min/max ACU settings"
  long_term: "Implement ACU monitoring and periodic capacity review"
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
