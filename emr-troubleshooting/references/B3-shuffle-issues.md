---
title: "B3 — Shuffle Issues"
description: "Diagnose Spark shuffle failures and performance problems"
status: active
severity: MEDIUM
triggers:
  - "shuffle failed"
  - "FetchFailedException"
  - "shuffle disk full"
  - "shuffle performance"
owner: devops-agent
objective: "Identify and resolve Spark shuffle issues on EMR"
context: "Shuffle operations redistribute data across executors. Failures occur due to disk space, network issues, executor loss, or memory pressure. FetchFailedException is the most common shuffle error. Shuffle data is stored on local disks."
---

## Phase 1 — Triage

MUST:
- Check for FetchFailedException in Spark logs
- Verify local disk space on cluster nodes
- Check for executor failures during shuffle
- Review Spark UI shuffle read/write metrics

SHOULD:
- Check shuffle partition count vs data volume
- Verify EBS volume configuration for shuffle storage
- Check network throughput between nodes
- Review spark.shuffle.compress and spark.shuffle.spill settings

MAY:
- Check for data skew causing uneven shuffle
- Review Spark external shuffle service status

## Phase 2 — Remediate

MUST:
- Ensure sufficient local disk space for shuffle data
- Fix executor stability issues causing shuffle fetch failures
- Adjust shuffle partition count for data volume

SHOULD:
- Add EBS volumes for additional shuffle storage
- Enable external shuffle service for resilience
- Optimize data partitioning to reduce shuffle size

MAY:
- Use broadcast joins to avoid shuffle for small tables
- Implement bucketing for frequently joined datasets

## Common Issues

- symptoms: "FetchFailedException during shuffle"
  diagnosis: "Executor lost during shuffle, shuffle data unavailable."
  resolution: "Enable external shuffle service. Fix executor OOM. Add retry."

- symptoms: "No space left on device during shuffle"
  diagnosis: "Local disk full from shuffle data."
  resolution: "Add EBS volumes. Reduce shuffle partitions. Increase disk size."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add EBS volumes for shuffle storage | YELLOW | Adds storage — cost increase, requires node restart |
| Enable external shuffle service | YELLOW | Changes shuffle behavior — test before production |
| Adjust shuffle partition count | GREEN | Configuration tuning — applied on next run |
| Use broadcast joins to avoid shuffle | GREEN | Query optimization — non-destructive |
| Implement bucketing for datasets | YELLOW | Changes data layout — requires data rewrite |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Shuffle failures cause repeated job failures affecting SLA
- Disk space issues risk data loss on HDFS

## Data Sensitivity

- **Classification: HIGH**
- Shuffle data may contain intermediate query results with sensitive data
- Disk usage patterns reveal data processing volumes
- Spark UI shuffle metrics expose data distribution and query patterns
- Local disk contents may include spilled data from memory

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest deleting shuffle data directories while jobs are running
- **NEVER** suggest reducing EBS volumes on running nodes

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Added EBS volumes | Detach added EBS volumes if no longer needed |
| Enabled external shuffle service | Disable external shuffle service and revert to default |
| Changed shuffle partition count | Revert spark.sql.shuffle.partitions to previous value |
| Implemented bucketing | Rewrite data without bucketing if it causes issues |

## Output Format

```yaml
root_cause: "shuffle_issue — <specific_cause>"
evidence:
  - type: shuffle_errors
    content: "<shuffle error details>"
  - type: disk_usage
    content: "<local disk utilization>"
severity: MEDIUM
mitigation:
  immediate: "Fix disk space or executor stability"
  long_term: "Configure adequate storage and enable external shuffle service"
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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"
